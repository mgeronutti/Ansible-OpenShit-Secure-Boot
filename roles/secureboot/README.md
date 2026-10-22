# Role: secureboot

Ativa ou desativa o Secure Boot de uma VM do OpenShift Virtualization,
desligando-a quando necessário e restaurando o estado de energia original.

Depende dos fatos produzidos pela role `common` (`common_vm_name`,
`common_vm_namespace`, `common_secureboot_desired`).

| Arquivo | Etapa |
|---|---|
| `tasks/main.yml` | Orquestração, `block`/`rescue` (religa a VM em caso de erro) |
| `tasks/gather_state.yml` | 1. Lê VM, VMI e preference; calcula se precisa alterar |
| `tasks/power_off.yml` | 2. Desliga (`runStrategy: Halted` ou `running: false`) e aguarda |
| `tasks/configure.yml` | 3. Grava `efi.secureBoot` (+ `smm` ao ativar) e confere |
| `tasks/power_on.yml` | 4. Religa e restaura o `runStrategy`/`running` original |
| `tasks/report.yml` | 5. Resumo |

| Variável (defaults) | Padrão | Descrição |
|---|---|---|
| `secureboot_wait_timeout` | `900` | Segundos máximos para desligar/ligar |
| `secureboot_wait_delay` | `10` | Intervalo entre verificações |

Explicação tarefa por tarefa em [docs/fluxo-detalhado.md](../../docs/fluxo-detalhado.md#36-role-secureboot).
