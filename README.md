# Ansible – Secure Boot de VMs no OpenShift Virtualization

Automação, executada pelo **Ansible Automation Platform (AAP)**, que **ativa ou
desativa o Secure Boot** de uma máquina virtual do OpenShift Virtualization e
**preserva o estado de energia** da VM:

1. Identifica a VM (hoje com nome fixo `win11-amber-heron-89`; no futuro, vinda do sistema interno).
2. Verifica se a VM está ligada ou desligada.
3. Se estiver ligada, desliga a VM.
4. Ativa ou desativa o Secure Boot, conforme a escolha do analista no survey do AAP.
5. Se a VM estava ligada, religa. Se estava desligada, ela continua desligada.

Se o Secure Boot **já estiver** no estado pedido, a VM não é desligada nem alterada.

| Documento | Conteúdo |
|---|---|
| [docs/aap-configuracao.md](docs/aap-configuracao.md) | **Cadastro no AAP passo a passo:** projeto, credencial, EE, inventário, job template, survey e permissões |
| [docs/fluxo-detalhado.md](docs/fluxo-detalhado.md) | Explicação de cada arquivo e de cada tarefa do código |

---

## Estrutura do projeto

```
.
├── ansible.cfg                     # Configuração do Ansible (roles, collections, saída)
├── collections/
│   └── requirements.yml            # Collections (kubernetes.core), instaladas pelo AAP na sincronização
├── requirements.txt                # Biblioteca Python (kubernetes), usada no Execution Environment
├── execution-environment.yml       # Definição do EE para o ansible-builder (se o EE padrão não atender)
├── playbooks/
│   └── secureboot.yml              # Playbook principal (o Job Template aponta para ele)
├── inventories/                    # Usados em execução LOCAL. No AAP vale o inventário do template
│   ├── development/
│   │   ├── hosts.yml               # localhost (a automação só fala com a API)
│   │   └── group_vars/all/
│   │       ├── openshift.yml       # Conexão (vazio = credencial do AAP / kubeconfig)
│   │       ├── vm.yml              # vm_name, vm_namespace, secureboot_action
│   │       └── vault.yml.example   # Token para testes locais (não usado no AAP)
│   └── production/                 # Mesmo layout
├── roles/
│   ├── common/                     # Define a VM alvo e valida a entrada do analista
│   ├── openshift_prerequisites/    # Testa a conexão e o OpenShift Virtualization
│   └── secureboot/                 # Estado, desligar, alterar, religar, resumo
│       ├── defaults/main.yml       # Tempos de espera (ajustáveis)
│       ├── vars/main.yml           # Constantes (versões de API)
│       └── tasks/
│           ├── main.yml            # Orquestração + recuperação em caso de erro
│           ├── gather_state.yml    # 1. Lê a VM: ligada? firmware? Secure Boot atual?
│           ├── power_off.yml       # 2. Desliga
│           ├── configure.yml       # 3. Grava o Secure Boot e confere
│           ├── power_on.yml        # 4. Religa e restaura o modo de energia original
│           └── report.yml          # 5. Resumo
├── files/
│   ├── rbac-secureboot.yml         # ServiceAccount + permissões mínimas no OpenShift
│   └── aap-survey-secureboot.json  # Survey pronto para importar no Job Template
└── docs/
```

---

## Execução pelo AAP

Resumo da configuração (o detalhamento está em [docs/aap-configuracao.md](docs/aap-configuracao.md)):

| Objeto no AAP | Configuração |
|---|---|
| **Projeto** | Este repositório Git. A sincronização instala `collections/requirements.yml` |
| **Credencial** | Tipo *OpenShift or Kubernetes API Bearer Token*, com a URL da API, o token da ServiceAccount (`files/rbac-secureboot.yml`) e a CA |
| **Execution Environment** | Precisa ter `kubernetes.core` e a lib Python `kubernetes`. Se o EE padrão não tiver, use o `execution-environment.yml` |
| **Inventário** | Apenas `localhost` com `ansible_connection: local` |
| **Job Template** | Playbook `playbooks/secureboot.yml`, *Concurrent jobs* desmarcado, *Timeout* de 1800 s |
| **Survey** | `secureboot_action` (enable/disable), `vm_name`, `vm_namespace`, a partir de `files/aap-survey-secureboot.json` |
| **Permissão** | Papel *Execute* no Job Template para o time de analistas |

**Como a conexão funciona no AAP:** a credencial entrega URL, token e
certificado ao container por variáveis de ambiente (`K8S_AUTH_*`), e a
collection kubernetes.core lê essas variáveis automaticamente. O playbook não
guarda nenhum segredo, e o analista nunca vê o token.

### Parâmetros (survey)

| Variável | Obrigatória | Padrão | Descrição |
|---|---|---|---|
| `secureboot_action` | **sim** | – | `enable` (ativar) ou `disable` (desativar) |
| `vm_name` | sim | `win11-amber-heron-89` | Nome da VM no OpenShift |
| `vm_namespace` | não | vazio | Namespace da VM. Se vazio, a VM é procurada em todos os namespaces |
| `secureboot_wait_timeout` | não | `900` | Segundos máximos para aguardar a VM desligar/ligar (pode ir em *Extra variables* do template) |
| `secureboot_wait_delay` | não | `10` | Intervalo, em segundos, entre as verificações |

### Exemplo do resumo exibido no job

```
VM ................: vms/win11-amber-heron-89
Ação solicitada ...: ATIVAR Secure Boot
Secure Boot antes .: DESATIVADO
Secure Boot agora .: ATIVADO
Alterado ..........: SIM
Energia ...........: estava ligada, foi desligada e religada
```

O histórico de cada execução (quem rodou, quando, parâmetros e saída) fica em
**Automation Execution > Jobs**.

---

## Execução local (para testes e desenvolvimento)

```bash
pip install ansible-core -r requirements.txt
ansible-galaxy collection install -r collections/requirements.yml

oc login https://api.<cluster-dev>:6443
ansible-playbook playbooks/secureboot.yml -e secureboot_action=enable
ansible-playbook playbooks/secureboot.yml -e secureboot_action=disable -e vm_namespace=meu-projeto --diff
```

Para autenticar sem `oc login`, preencha `openshift_api_url` em
`inventories/<ambiente>/group_vars/all/openshift.yml` e coloque o token em um
`vault.yml` criptografado (veja `vault.yml.example`). Depois execute com `--ask-vault-pass`.

---

## Comportamentos de segurança

- **Sem alterações desnecessárias:** se o Secure Boot já estiver como pedido, nada é feito.
- **Validação antes de tocar na VM:** ação inválida, nome inválido, VM inexistente ou
  nome duplicado em vários namespaces interrompem a execução antes de qualquer alteração.
- **VM com firmware BIOS:** a ativação é recusada **antes** de desligar a VM. Converter
  BIOS em EFI impede o Windows instalado de iniciar.
- **Recuperação:** se algo falhar depois que a VM foi desligada, o playbook tenta
  religá-la antes de encerrar com o erro (o job aparece como *Failed* no AAP).
- **Modo de energia preservado:** `runStrategy` (Always, RerunOnFailure, Manual…) ou
  `spec.running` voltam exatamente ao valor original.
- **Desligamento gracioso:** o Windows recebe um sinal ACPI e encerra normalmente.
  O desligamento só é forçado após o `terminationGracePeriodSeconds` da VM.
- **Sem execução simultânea:** com *Concurrent jobs* desmarcado, dois analistas não
  alteram a mesma VM ao mesmo tempo.

---

## Problemas comuns

| Mensagem | Causa / solução |
|---|---|
| `Informe a ação desejada com -e secureboot_action=...` | Survey desativado ou sem a variável `secureboot_action` |
| `couldn't resolve module/action 'kubernetes.core...'` | O EE não tem a collection e o projeto não sincronizou `collections/requirements.yml` |
| `Não foi possível conectar na API do OpenShift` | Credencial não associada ao template, token expirado, CA ausente ou nó de execução sem acesso à porta 6443 |
| `A API kubevirt.io/v1 ... não foi encontrada` | O OpenShift Virtualization não está instalado ou a ServiceAccount não tem acesso |
| `VM '...' não encontrada` | Nome errado ou falta de permissão no namespace |
| `Existem N VMs chamadas ...` | Preencha o campo *Namespace da VM* no survey |
| `A VM ... usa firmware BIOS` | Secure Boot exige EFI. A VM não foi alterada |
| Timeout em `Aguardar a VM desligar` | O Windows demorou a desligar. Aumente `secureboot_wait_timeout` (e o *Timeout* do template) |

Mais casos específicos do AAP em [docs/aap-configuracao.md](docs/aap-configuracao.md#problemas-comuns-no-aap).

---

## Integração futura com o sistema interno

Há dois caminhos possíveis:

1. **O sistema interno chama a API do AAP** e passa `vm_name` e `secureboot_action`.
   Nenhuma mudança no código é necessária (exemplo em `docs/aap-configuracao.md`, seção 10).
2. **O playbook consulta o sistema interno:** implemente a consulta em
   [roles/common/tasks/main.yml](roles/common/tasks/main.yml), onde há um exemplo comentado,
   e grave o resultado em `common_vm_name`.
