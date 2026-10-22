# Configuração no Ansible Automation Platform (AAP)

Passo a passo para cadastrar a automação de Secure Boot no AAP. Os nomes de
menu seguem o AAP 2.5 (**Automation Execution**). No AAP 2.4, os mesmos itens
ficam em **Resources** e **Administration**.

```
 Analista ──► Job Template "OpenShift | Secure Boot de VM"
                 │  Survey: secureboot_action, vm_name, vm_namespace
                 │
                 ├─ Projeto ............ repositório Git deste código
                 ├─ Execution Env. ..... container com kubernetes.core + python kubernetes
                 ├─ Inventário ......... localhost (a automação só fala com a API)
                 └─ Credencial ......... OpenShift or Kubernetes API Bearer Token
                                              │
                                              ▼
                                   API do OpenShift (HTTPS 6443)
```

> **Rede:** quem acessa a API do OpenShift é o container do EE, que roda no nó
> de execução do AAP. Esse nó precisa alcançar `https://api.<cluster>:6443`.

---

## 1. Usuário técnico no OpenShift (ServiceAccount)

Crie uma ServiceAccount com permissões mínimas usando o arquivo do projeto:

```bash
oc new-project ansible-automation          # se ainda não existir
oc apply -f files/rbac-secureboot.yml
oc create token ansible-secureboot -n ansible-automation --duration=8760h
```

Guarde o token gerado: ele será usado na credencial (passo 3). Os tokens criados
com `oc create token` expiram, então registre a data de renovação. Se preferir um
token sem expiração, use um Secret do tipo `kubernetes.io/service-account-token`.

## 2. Projeto

**Automation Execution > Projects > Create project**

| Campo | Valor |
|---|---|
| Name | `OpenShift Secure Boot` |
| Source control type | Git |
| Source control URL | URL do repositório deste código |
| Source control branch | `main` (ou a branch de produção) |
| Source control credential | Credencial de leitura do Git, se o repositório for privado |
| Options | ✅ *Update revision on launch* (recomendado) |

Na sincronização, o AAP instala as collections listadas em
`collections/requirements.yml` (kubernetes.core).

## 3. Credencial do OpenShift

**Automation Execution > Infrastructure > Credentials > Create credential**

| Campo | Valor |
|---|---|
| Name | `OpenShift PRD - ansible-secureboot` |
| Credential type | **OpenShift or Kubernetes API Bearer Token** |
| OpenShift or Kubernetes API Endpoint | `https://api.<cluster>:6443` |
| API authentication bearer token | token gerado no passo 1 |
| Verify SSL | ✅ marcado |
| Certificate Authority data | CA da API, se o certificado não for de uma CA pública. Obtenha com `oc get cm kube-root-ca.crt -n default -o jsonpath='{.data.ca\.crt}'` |

Ao executar o job, o AAP entrega esses dados ao playbook pelas variáveis de
ambiente `K8S_AUTH_HOST`, `K8S_AUTH_API_KEY`, `K8S_AUTH_VERIFY_SSL` e
`K8S_AUTH_SSL_CA_CERT`. A collection kubernetes.core lê essas variáveis
automaticamente. **Não** é preciso usar vault nem preencher `openshift_api_url`.

Para ter clusters diferentes (DEV/PRD), crie uma credencial e um Job Template
para cada um.

## 4. Execution Environment

O EE precisa ter a collection **kubernetes.core** e a biblioteca Python **kubernetes**.

1. **Primeiro verifique o EE "supported" do seu AAP** (ex.: `ee-supported-rhel9`).
   Normalmente ele já traz a kubernetes.core e dispensa uma imagem própria.
   Para conferir, execute o Job Template uma vez: se a etapa
   `Prerequisitos | Consultar informações do cluster` passar, o EE está pronto.
2. Se o EE não tiver essas dependências, construa uma imagem com o arquivo
   [execution-environment.yml](../execution-environment.yml):
   ```bash
   pip install ansible-builder
   podman login registry.redhat.io
   ansible-builder build -t <registry>/<org>/ee-openshift-secureboot:1.0 -f execution-environment.yml
   podman push <registry>/<org>/ee-openshift-secureboot:1.0
   ```
   Depois cadastre a imagem em **Automation Execution > Infrastructure > Execution Environments**.

## 5. Inventário

**Automation Execution > Infrastructure > Inventories > Create inventory**

- Name: `Localhost – API OpenShift`
- Adicione o host `localhost` com as variáveis:
  ```yaml
  ansible_connection: local
  ansible_python_interpreter: "{{ ansible_playbook_python }}"
  ```

O playbook só conversa com a API, então `localhost` aqui é o próprio container
do EE. Não é preciso cadastrar as VMs como hosts.

> Alternativa: crie um inventário com origem **Sourced from a Project** apontando
> para `inventories/production/hosts.yml`. Os valores de `group_vars` vêm
> vazios, para não sobrepor a credencial.

## 6. Job Template

**Automation Execution > Templates > Create template > Create job template**

| Campo | Valor |
|---|---|
| Name | `OpenShift \| Secure Boot de VM` |
| Job type | Run |
| Inventory | `Localhost – API OpenShift` |
| Project | `OpenShift Secure Boot` |
| Playbook | `playbooks/secureboot.yml` |
| Execution environment | O EE do passo 4 |
| Credentials | `OpenShift PRD - ansible-secureboot` |
| Verbosity | 0 (Normal). Use 1 ou 2 para diagnóstico |
| Timeout | `1800` (deve ser maior que o tempo de desligar + ligar a VM) |
| Options | ❌ *Concurrent jobs* desmarcado, para evitar dois analistas mexendo na mesma VM ao mesmo tempo |

## 7. Survey (formulário do analista)

No Job Template, abra a aba **Survey** e crie as perguntas abaixo.

| Pergunta | Variável | Tipo | Obrigatória | Padrão |
|---|---|---|---|---|
| Ação de Secure Boot | `secureboot_action` | Multiple choice: `enable`, `disable` | Sim | – |
| Nome da VM | `vm_name` | Text (1–63) | Sim | `win11-amber-heron-89` |
| Namespace da VM | `vm_namespace` | Text (0–63) | Não | vazio |

Depois **ative** o survey (botão *Survey enabled*).

Também é possível importar o arquivo pronto
[files/aap-survey-secureboot.json](../files/aap-survey-secureboot.json) pela API:

```bash
curl -X POST "https://<aap>/api/controller/v2/job_templates/<ID>/survey_spec/" \
  -H "Authorization: Bearer <token-aap>" -H "Content-Type: application/json" \
  -d @files/aap-survey-secureboot.json
# AAP 2.4: use /api/v2/job_templates/<ID>/survey_spec/
```

As respostas do survey são passadas como *extra vars* e, por isso, têm
prioridade sobre qualquer valor definido em `group_vars`.

## 8. Permissões no AAP

No Job Template, aba **User access** / **Team access**, dê ao time de analistas
o papel **JobTemplate Execute**. Assim eles conseguem executar a automação, mas
não editar o template, o projeto ou a credencial (o token nunca fica visível).

## 9. Execução e resultado

1. O analista clica em **Launch**, preenche o survey e confirma.
2. A saída do job mostra o painel `Estado | Exibir o diagnóstico da VM` e, no
   final, `Resumo | Resultado da execução`.
3. O histórico fica em **Automation Execution > Jobs**, com usuário, data,
   parâmetros (extra vars) e saída completa. Isso substitui o arquivo de log local.

## 10. Recomendações opcionais

- **Aprovação para produção:** coloque o Job Template em um **Workflow** com um
  nó de *Approval* antes da execução.
- **Notificações:** associe um *Notification Template* (e-mail, Teams, Slack)
  para os eventos de sucesso e falha do job.
- **Integração com o sistema interno:** há duas formas.
  - O sistema interno chama a API do AAP passando a VM:
    ```bash
    curl -X POST "https://<aap>/api/controller/v2/job_templates/<ID>/launch/" \
      -H "Authorization: Bearer <token-aap>" -H "Content-Type: application/json" \
      -d '{"extra_vars": {"secureboot_action": "enable", "vm_name": "win11-amber-heron-89"}}'
    ```
  - Ou o playbook consulta o sistema interno, no ponto marcado em
    `roles/common/tasks/main.yml`. Nesse caso, guarde a credencial desse sistema
    em uma *Custom Credential Type* do AAP.

## Problemas comuns no AAP

| Sintoma | Causa provável |
|---|---|
| `couldn't resolve module/action 'kubernetes.core.k8s_info'` | O EE não tem a collection e o projeto não sincronizou `collections/requirements.yml` |
| `Failed to import the required Python library (kubernetes)` | O EE não tem a biblioteca Python. Construa o EE do passo 4 |
| `Não foi possível conectar na API do OpenShift` | Credencial não associada ao template, token expirado ou nó de execução sem rota para a porta 6443 |
| `certificate verify failed` | Preencha *Certificate Authority data* na credencial |
| `Forbidden` / `VM ... não encontrada` | A ServiceAccount não tem permissão no namespace da VM (veja `files/rbac-secureboot.yml`) |
| Job encerrado por timeout | Aumente o *Timeout* do Job Template e/ou `secureboot_wait_timeout` |
