# vault_demo
helm install vault hashicorp/vault     --set "global.openshift=true"     --set "server.dev.enabled=true" --set="injector.enabled=false"

Note: edit statefulset and update image, default in chart is old. 


    1  vault auth enable kubernetes
    2  vault write auth/kubernetes/config token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    3  vault kv put secret/vault-demo-secret1 username="phil" password="notverysecure"
    4  vault kv get secret secret/vault-demo-secret1
    5  vault kv get secret/vault-demo-secret1
