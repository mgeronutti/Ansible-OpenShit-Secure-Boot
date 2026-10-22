# Role: openshift_prerequisites

Verificações somente-leitura executadas antes de alterar qualquer VM:

1. Biblioteca Python `kubernetes` disponível
2. Conexão e autenticação com a API do OpenShift (`k8s_cluster_info`)
3. API `kubevirt.io/v1` (OpenShift Virtualization) presente

Usa os parâmetros de conexão injetados pelo `module_defaults` do playbook
(`openshift_api_url`, `openshift_api_key`, `openshift_validate_certs`) ou, se
vazios, o kubeconfig atual. Detalhes em
[docs/fluxo-detalhado.md](../../docs/fluxo-detalhado.md#35-role-openshift_prerequisites-rolesopenshift_prerequisitestasksmainyml).
