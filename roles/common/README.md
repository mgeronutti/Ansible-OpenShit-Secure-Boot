# Role: common

Define a VM alvo e valida a entrada do analista **antes** de qualquer acesso ao cluster.

| Entrada | Descrição |
|---|---|
| `vm_name` | Nome da VM (hoje fixo em `group_vars/all/vm.yml`) |
| `vm_namespace` | Namespace (`""` = procurar em todos) |
| `secureboot_action` | `enable` ou `disable` (obrigatório) |

| Saída (fatos) | Descrição |
|---|---|
| `common_vm_name` / `common_vm_namespace` | VM alvo já validada |
| `common_secureboot_desired` | `true` (ativar) / `false` (desativar) |

A integração com o sistema interno deve ser feita em `tasks/main.yml`, no ponto
indicado pelo comentário "INTEGRAÇÃO FUTURA". Detalhes em
[docs/fluxo-detalhado.md](../../docs/fluxo-detalhado.md#34-role-common-rolescommontasksmainyml).
