# Fluxo detalhado – Secure Boot de VMs no OpenShift

Este documento explica **o que cada arquivo faz, em que ordem e por quê**.
Leia junto com o código: cada tarefa citada aqui tem o mesmo nome no arquivo
`.yml` (por exemplo, `Estado | Localizar a VirtualMachine`) e também aparece com
esse nome na saída do `ansible-playbook`.

---

## 1. Conceitos necessários

### 1.1 Onde o Ansible roda

O playbook **não acessa a VM por SSH ou WinRM**. Toda a interação ocorre pela
**API do OpenShift**, do mesmo jeito que o comando `oc` ou o console web. Por isso
o inventário contém apenas `localhost`. Em produção, quem executa é o **Ansible
Automation Platform**: o playbook roda dentro do container do *Execution
Environment* (EE) e envia requisições HTTPS para o cluster. Nos testes locais,
ele roda na sua máquina da mesma forma.

### 1.1.1 De onde vêm os dados de conexão

| Execução | URL / token / certificado |
|---|---|
| **AAP** | Credencial *OpenShift or Kubernetes API Bearer Token* do Job Template. O AAP exporta as variáveis `K8S_AUTH_HOST`, `K8S_AUTH_API_KEY`, `K8S_AUTH_VERIFY_SSL` e `K8S_AUTH_SSL_CA_CERT`, e a collection lê essas variáveis sozinha |
| **Local** | `~/.kube/config` (sessão do `oc login`) ou `openshift_api_url` + token no vault |

As variáveis do analista (`secureboot_action`, `vm_name`, `vm_namespace`)
chegam pelo **Survey** do Job Template como *extra vars*, que têm a maior
prioridade do Ansible.

Os módulos usados são da collection `kubernetes.core`:

| Módulo | Para que serve aqui |
|---|---|
| `kubernetes.core.k8s_cluster_info` | Testa a conexão e lista as APIs que o cluster oferece |
| `kubernetes.core.k8s_info` | **Lê** objetos (VM, VMI, preference). Nunca altera nada |
| `kubernetes.core.k8s` | **Altera** objetos. Usado sempre com `state: patched` |

### 1.2 VirtualMachine × VirtualMachineInstance

| Objeto | O que é | Quando existe |
|---|---|---|
| **VirtualMachine (VM)** | A "ficha" da máquina: CPU, memória, discos, firmware | Sempre, ligada ou não |
| **VirtualMachineInstance (VMI)** | A máquina **em execução** | Só enquanto está ligada |

Regra usada pelo playbook: **se existe VMI, a VM está ligada**.

### 1.3 Como uma VM é ligada e desligada

A VM não tem um botão "ligar". Ela tem um campo que diz **como deveria
estar**, e o OpenShift age para cumprir isso. Esse campo pode ser:

| Campo | Valores | Observação |
|---|---|---|
| `spec.runStrategy` (atual) | `Always`, `RerunOnFailure`, `Manual`, `Halted`, `Once` | `Halted` = desligada |
| `spec.running` (antigo) | `true` / `false` | Ainda aceito em VMs antigas |

Uma VM usa **um ou outro**, nunca os dois. O playbook detecta qual campo a VM
usa e altera somente esse campo.

### 1.4 Onde fica o Secure Boot

```yaml
spec:
  template:
    spec:
      domain:
        features:
          smm:
            enabled: true        # obrigatório quando secureBoot = true
        firmware:
          bootloader:
            efi:
              secureBoot: true   # true = ativado / false = desativado
              persistent: true   # (opcional) guarda variáveis EFI do Windows
```

Regras do KubeVirt que o playbook respeita:

- `efi: {}` **sem** `secureBoot` significa Secure Boot **ativado** (é o padrão).
- `bios: {}`, ou nenhum bootloader, significa **sem** Secure Boot.
- `secureBoot: true` exige `features.smm.enabled: true`. Sem isso, a API recusa a alteração.
- VMs criadas a partir de **InstanceType** (ex.: preference `windows.11`)
  podem não ter `firmware` na própria VM e herdá-lo da *preference*.

### 1.5 Por que é preciso desligar

O firmware só é lido quando a máquina **inicia**. Mudar o Secure Boot com a VM
ligada não teria efeito até o próximo boot. Por isso a sequência é
desligar → alterar → religar.

---

## 2. Visão geral do fluxo

```
ansible-playbook playbooks/secureboot.yml -e secureboot_action=enable|disable
│
├─ role common ─────────────────────────── (nenhum acesso ao cluster)
│    ├─ define a VM alvo (nome fixo / futuro sistema interno)
│    ├─ valida o nome (padrão DNS-1123)
│    └─ valida a ação (enable|disable) ──────────── inválida? → PARA
│
├─ role openshift_prerequisites ────────── (somente leitura)
│    ├─ conecta e autentica na API ─────────────── falhou?   → PARA
│    └─ confirma que kubevirt.io/v1 existe ──────── não?      → PARA
│
└─ role secureboot
     ├─ gather_state.yml (somente leitura)
     │    ├─ encontra a VM (exatamente 1) ────────── 0 ou >1? → PARA
     │    ├─ descobre o controle de energia (runStrategy / running)
     │    ├─ VMI existe? → was_running = sim/não
     │    ├─ descobre o firmware (VM → preference → padrão BIOS)
     │    ├─ compara Secure Boot atual × desejado
     │    └─ BIOS + enable? ───────────────────────── sim?     → PARA (VM intocada)
     │
     ├─ já está no estado desejado? ─── sim → só exibe a mensagem e vai ao resumo
     │
     ├─ BLOCK ──────────────────────────────────────────────────────────┐
     │    ├─ power_off.yml   (só se was_running)  desliga e espera        │
     │    ├─ configure.yml   grava secureBoot (+SMM) e relê para conferir │
     │    └─ power_on.yml    (só se was_running)  religa e espera         │
     ├─ RESCUE (qualquer erro dentro do block) ─────────────────────────┤
     │    ├─ guarda o erro original                                       │
     │    ├─ power_on.yml (só se was_running): tenta religar             │
     │    └─ falha exibindo o erro original                               │
     │                                                                    ┘
     └─ report.yml  resumo final
```

---

## 3. Arquivo por arquivo

### 3.1 `ansible.cfg`

Configurações lidas automaticamente (o AAP também usa este arquivo, porque ele
fica na raiz do projeto):

- `inventory`: inventário usado **só em execução local** (development). No AAP
  vale o inventário escolhido no Job Template.
- `collections_path = ./collections`: collections instaladas localmente com
  `ansible-galaxy collection install -r collections/requirements.yml`. No AAP,
  a sincronização do projeto lê o mesmo `collections/requirements.yml`.
- `log_path` fica comentado: no AAP a saída de cada job já é armazenada.
- `result_format = yaml`: saída mais legível (também na tela do job no AAP).

### 3.2 `inventories/<ambiente>/`

- `hosts.yml`: só `localhost` com `ansible_connection: local`.
Usados em execução **local** ou quando o AAP importa o inventário do projeto
(*Sourced from a Project*).

- `group_vars/all/openshift.yml`: endereço da API, token e validação TLS.
  Fica **vazio de propósito**: assim a credencial do AAP (ou o kubeconfig,
  localmente) é usada. Um valor preenchido aqui teria prioridade sobre a credencial.
- `group_vars/all/vm.yml`: `vm_name`, `vm_namespace` e `secureboot_action`.
  Valores passados com `-e` na linha de comando **sempre** têm prioridade.
- `group_vars/all/vault.yml.example`: modelo do arquivo de segredos.

> As pastas `group_vars/` e `host_vars/` na **raiz** do projeto não são lidas
> pelo Ansible, porque ele só procura essas pastas ao lado do inventário ou do
> playbook. Use as pastas que ficam dentro de `inventories/<ambiente>/`.

### 3.3 `playbooks/secureboot.yml`

- `hosts: localhost` / `connection: local`: roda na máquina local.
- `gather_facts: false`: não coleta dados do sistema local, que não são usados.
- `module_defaults` → `group/kubernetes.core.k8s`: aplica `host`, `api_key`
  e `validate_certs` a **todos** os módulos `kubernetes.core.*`. Assim nenhuma
  tarefa precisa repetir os dados de conexão. O filtro `default(omit, ...)`
  remove o parâmetro quando ele está vazio ou indefinido. Nesse caso, o módulo
  usa as variáveis `K8S_AUTH_*` da credencial do AAP ou o kubeconfig local.
- `roles`: executa `common`, depois `openshift_prerequisites` e por último `secureboot`.

### 3.4 Role `common` ([roles/common/tasks/main.yml](../roles/common/tasks/main.yml))

| Tarefa | O que faz |
|---|---|
| `Common \| Definir a VM alvo` | Copia `vm_name`/`vm_namespace` para `common_vm_name`/`common_vm_namespace`. **É aqui que a integração com o sistema interno vai entrar** (há um exemplo comentado no arquivo) |
| `Common \| Validar o nome e o namespace` | Garante o padrão do OpenShift: minúsculas, números e hífen, com até 63 caracteres |
| `Common \| Validar a ação de Secure Boot` | Aceita só `enable`/`disable`. Qualquer outro valor encerra o playbook **antes** de conectar no cluster |
| `Common \| Converter a ação em valor booleano` | `enable` vira `true` e `disable` vira `false`, gravado em `common_secureboot_desired` |
| `Common \| Exibir o que será feito` | Mostra a VM, o namespace e a ação |

### 3.5 Role `openshift_prerequisites` ([roles/openshift_prerequisites/tasks/main.yml](../roles/openshift_prerequisites/tasks/main.yml))

| Tarefa | O que faz |
|---|---|
| `Identificar o endereço da API` | Descobre a URL e de onde ela veio: variável `openshift_api_url`, credencial do AAP (`K8S_AUTH_HOST`) ou kubeconfig |
| `Exibir o destino da conexão` | Mostra a URL e a origem dela |
| `Validar o formato do endereço da API` | Antes de conectar, recusa URLs sem `https://` ou que ainda tenham o texto de exemplo (`api.<cluster>`) |
| `Consultar informações do cluster` | `k8s_cluster_info`: conecta, autentica e lista as APIs. Dentro de `block/rescue` |
| `Analisar o erro` / `Classificar a causa` (rescue) | Procura no erro técnico as assinaturas de problemas conhecidos e mostra um **diagnóstico** com a correção: DNS, CERTIFICADO, TOKEN, PERMISSÃO, REDE ou EXECUTION ENVIRONMENT |
| `Exibir versão do cluster` | Confirmação visual da conexão |
| `Confirmar que o OpenShift Virtualization está disponível` | Procura o tipo `VirtualMachine` na API `kubevirt.io/v1` |

### 3.6 Role `secureboot`

#### `defaults/main.yml` e `vars/main.yml`

- `secureboot_wait_timeout` (900s) e `secureboot_wait_delay` (10s) controlam as
  esperas. O timeout precisa ser **maior** que o `terminationGracePeriodSeconds`
  da VM, que é o tempo dado ao Windows para desligar sozinho.
- `secureboot_wait_retries` = timeout ÷ intervalo, ou seja, quantas vezes o estado é consultado.
- As versões de API ficam em `vars/` porque não devem ser alteradas pelo analista.

#### `tasks/main.yml`: orquestração

1. Importa `gather_state.yml`.
2. Se `secureboot_change_required` for **false**, exibe "já está no estado
   desejado" e pula para o resumo. A VM não é desligada.
3. Se for **true**, entra no `block`:
   - `power_off.yml`, somente se `secureboot_vm_was_running`
   - `configure.yml`
   - `power_on.yml`, somente se `secureboot_vm_was_running`
4. **`rescue`**: se qualquer tarefa do block falhar, o Ansible pula para cá.
   - Guarda a mensagem e o nome da tarefa que falhou (`ansible_failed_result`, `ansible_failed_task`).
   - Se a VM estava ligada, executa `power_on.yml` para religá-la.
   - Encerra com `fail`, mostrando o erro original. A execução fica marcada como falha.
5. Importa `report.yml`.

> `import_tasks` × `include_tasks`: o `import` é resolvido antes da execução
> e sempre roda. O `include` é avaliado na hora, o que permite usar `when` e
> reutilizar o mesmo arquivo, como o `power_on.yml` no block e no rescue.

#### `tasks/gather_state.yml`: etapa 1, levantamento (somente leitura)

| Tarefa | O que faz |
|---|---|
| `Localizar a VirtualMachine` | `k8s_info` com `field_selectors: metadata.name=<nome>`. Sem namespace, busca em todos |
| `Garantir que exatamente uma VM foi encontrada` | 0 = não existe ou sem permissão. Mais de 1 = mesmo nome em vários namespaces, e o analista deve informar `vm_namespace` |
| `Extrair a configuração da VM` | Guarda a VM em `secureboot_vm`, o namespace real, o status do console (`printableStatus`) e o bloco `bootloader` |
| `Identificar o controle de energia` | `secureboot_uses_running_field` (VM usa `spec.running`?) e `secureboot_original_run_strategy` (valor original, para restaurar depois) |
| `Verificar se existe uma instância em execução (VMI)` + `Registrar se a VM estava ligada` | Existe VMI → `secureboot_vm_was_running = true` |
| `Ler a preference (InstanceType)` | Só roda se a VM **não** declara bootloader e tem `spec.preference`. Lê a `VirtualMachineClusterPreference` (padrão) ou `VirtualMachinePreference` |
| `Assumir firmware padrão (BIOS)` | Ponto de partida: BIOS, sem Secure Boot |
| `Firmware definido na própria VM` | Se a VM tem `bootloader`: `efi` ou `bios` |
| `Firmware herdado da preference` | Usa `preferredEfi` (formato atual) ou `preferredUseEfi`/`preferredUseSecureBoot` (formato antigo) |
| `Calcular se a alteração é necessária` | `secureboot_current_enabled` = firmware EFI **e** `secureBoot`, que vale true quando ausente |
| `Comparar estado atual com o desejado` | `secureboot_change_required` = atual ≠ desejado |
| `Exibir o diagnóstico da VM` | Painel com tudo o que foi descoberto |
| `Bloquear ativação em VM com firmware BIOS` | Se for preciso ativar e a VM é BIOS, para **antes** de desligar. Converter BIOS em EFI impede o Windows instalado de iniciar |

#### `tasks/power_off.yml`: etapa 2, desligar

| Tarefa | O que faz |
|---|---|
| `Solicitar desligamento via spec.running=false` | Para VMs no formato antigo |
| `Solicitar desligamento via runStrategy=Halted` | Para VMs no formato atual |
| `Aguardar a VM desligar` | Consulta a VMI a cada `secureboot_wait_delay` s (`until`/`retries`/`delay`) até ela sumir. Se o tempo acabar, a tarefa falha e o `rescue` religa a VM |

**O que é `state: patched` com `merge_type: merge`?** É um *JSON Merge Patch*:
somente os campos enviados em `definition` são alterados e todo o resto da VM
permanece igual. Por exemplo, enviar `spec: {runStrategy: Halted}` muda apenas
esse campo.

O OpenShift desliga de forma **graciosa**: envia um sinal ACPI ao Windows, como
um toque rápido no botão de energia, e só força o desligamento depois de
`terminationGracePeriodSeconds`.

#### `tasks/configure.yml`: etapa 3, alterar o Secure Boot

| Tarefa | O que faz |
|---|---|
| `Montar a alteração do firmware` | Copia o EFI atual (preservando, por exemplo, `persistent: true`), troca só o `secureBoot` e, ao **ativar**, acrescenta `features.smm.enabled: true` |
| `Exibir o que será gravado` | Mostra o trecho exato enviado para a API |
| `Gravar o novo valor de Secure Boot na VM` | `k8s` com `state: patched` em `spec.template.spec.domain` |
| `Reler a VM para conferir` + `Validar o Secure Boot gravado` | Lê a VM de novo **no cluster** e confirma o valor. Isso garante que nenhum webhook ou operador desfez a alteração |

Exemplo do patch enviado ao **ativar**:

```yaml
spec:
  template:
    spec:
      domain:
        features:  { smm: { enabled: true } }
        firmware:  { bootloader: { efi: { secureBoot: true } } }
```

Ao **desativar**, o patch tem apenas o `firmware` com `secureBoot: false`. O SMM
continua como está, porque não interfere no boot.

#### `tasks/power_on.yml`: etapa 4, religar

Só roda se a VM estava ligada no início, ou no `rescue`.

| Tarefa | O que faz |
|---|---|
| `Definir o runStrategy usado para ligar` | `Always`, `RerunOnFailure` e `Once` voltam ao valor original. `Manual` (e qualquer outro) usa `Always` temporariamente |
| `Solicitar inicialização via spec.running=true` | Formato antigo |
| `Solicitar inicialização via runStrategy=...` | Formato atual |
| `Aguardar a VM ficar pronta` | Espera `printableStatus = Running` e `ready = true` |
| `Restaurar runStrategy original (Manual)` | Com a VM já ligada, grava `Manual` de novo. Com `Manual`, mudar o campo não desliga a VM |

> "Pronta" (`ready`) significa que o hypervisor iniciou a VM. O Windows pode
> ainda estar carregando. Se for preciso aguardar o sistema operacional, é
> possível acrescentar uma verificação do *guest agent* (`AgentConnected`).

#### `tasks/report.yml`: etapa 5, resumo

Mostra a VM, a ação, o Secure Boot antes e depois, se houve alteração e o que
aconteceu com a energia da VM.

---

## 4. Matriz de cenários

| Estado inicial | Secure Boot atual | Ação | Resultado |
|---|---|---|---|
| Ligada | Desativado | enable | Desliga → ativa (+SMM) → religa |
| Ligada | Ativado | disable | Desliga → desativa → religa |
| Desligada | Desativado | enable | Ativa → **continua desligada** |
| Desligada | Ativado | disable | Desativa → **continua desligada** |
| Qualquer | = ação | – | Nada é feito, e a VM não é desligada |
| Qualquer, BIOS | Desativado | enable | Falha **sem desligar** a VM |
| Ligada | – | – (erro no meio) | `rescue` religa a VM e mostra o erro |

---

## 5. Variáveis produzidas durante a execução

| Variável | Origem | Significado |
|---|---|---|
| `common_vm_name` / `common_vm_namespace` | common | VM alvo |
| `common_secureboot_desired` | common | `true` = ativar, `false` = desativar |
| `secureboot_vm` | gather_state | Objeto completo da VM lido do cluster |
| `secureboot_vm_namespace` | gather_state | Namespace real da VM |
| `secureboot_vm_was_running` | gather_state | A VM estava ligada no início? |
| `secureboot_uses_running_field` | gather_state | A VM usa `spec.running` (formato antigo)? |
| `secureboot_original_run_strategy` | gather_state | runStrategy original, para restaurar |
| `secureboot_firmware_type` | gather_state | `efi` ou `bios` |
| `secureboot_firmware_source` | gather_state | De onde veio o firmware (VM, preference, padrão) |
| `secureboot_current_efi` | gather_state | Bloco EFI atual (base do patch) |
| `secureboot_current_enabled` | gather_state | Secure Boot ativo hoje? |
| `secureboot_change_required` | gather_state | É preciso alterar? |
| `secureboot_domain_patch` | configure | Trecho enviado para a API |

---

## 6. Dicas de execução

| Objetivo | Linha de comando | AAP (Job Template) |
|---|---|---|
| Mais detalhes na saída | `-v` / `-vvv` | *Verbosity* 1 ou 3 |
| Ver o antes/depois de cada alteração na VM | `--diff` | Opção *Show changes* |
| Simular sem alterar nada | `--check` | *Job type* = **Check** |

No modo Check, as alterações não são gravadas e as esperas são puladas. Como a
VM não é desligada de verdade, o resultado serve apenas como prévia.

Outras observações:

- `--start-at-task "Estado | Exibir o diagnóstico da VM"` **não** é recomendado:
  as etapas dependem das variáveis calculadas antes delas.
