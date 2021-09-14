vault write auth/kubernetes/role/exsecretsdemo bound_service_account_names=es-kubernetes-external-secrets bound_service_account_namespaces=external-secrets policies=pmodemo ttl=60m



# vault_demo
helm install vault hashicorp/vault     --set "global.openshift=true"     --set "server.dev.enabled=true" --set="injector.enabled=false"

Note: edit statefulset and update image, default in chart is old. 


    1  vault auth enable kubernetes
    2  vault write auth/kubernetes/config token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    3  vault kv put secret/vault-demo-secret1 username="phil" password="notverysecure"
    4  vault kv get secret secret/vault-demo-secret1
    5  vault kv get secret/vault-demo-secret1

   19  vault policy write pmodemo - <<EOF
path "secret/data/vault-demo-secret1" {
  capabilities = ["read"]
}
                                          
                                          

   21  vault write auth/kubernetes/role/pmodemo bound_service_account_names=vault bound_service_account_namespaces=vault-demo policies=pmodemo ttl=60m
   
https://github.com/external-secrets/kubernetes-external-secrets/issues/721
$ vault write auth/kubernetes/config \
        token_reviewer_jwt="$SA_JWT_TOKEN" \
        kubernetes_host="$K8S_HOST" \
        kubernetes_ca_cert="$SA_CA_CRT" \
        disable_iss_validation=true
    12  vault write auth/kubernetes/config kubernetes_host=https://172.30.0.1:443 disable_iss_validation=true
   13  vault read auth/kubernetes/config
                                          
                                          
                                          
    9  OCP_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
   11  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": ""}' http://172.30.46.242:8200/v1/auth/kubernetes/login

                                          
                                          unsupported protocol scheme ""
                                          
                                          sh-4.4$ vault write auth/kubernetes/config disable_iss_validation=true kubernetes_host="https://172.30.0.1:443"
Success! Data written to: auth/kubernetes/config
sh-4.4$ OCP_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)sh-4.4$ curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": ""}' http://172.30.46.242:8200/v1/auth/kubernetes/login
{"errors":["missing role"]}
                                          
                                          
sh-4.4$ curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "pmodemo"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
{"request_id":"27734593-7662-b022-4a75-1f94bdcfb67a","lease_id":"","renewable":false,"lease_duration":0,"data":null,"wrap_info":null,"warnings":null,"auth":{"client_token":"s.kAWioc9w8tP2UtgyPod1xf1p","accessor":"3k2pu08LSRqSgbhUez6lGAzB","policies":["default","pmodemo"],"token_policies":["default","pmodemo"],"metadata":{"role":"pmodemo","service_account_name":"vault","service_account_namespace":"vault-demo","service_account_secret_name":"","service_account_uid":"8b761230-e6e8-447f-bf87-b561f0646f47"},"lease_duration":3600,"renewable":true,"entity_id":"657d8d0a-9108-dbad-56eb-eea4e4cc3ec4","token_type":"service","orphan":true}}
                                          
                                          
                                          vault write auth/kubernetes/config token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt issuer=https://kubernetes.default.svc
                                          
                                          
                                          
                                          
                                         
      1  vault auth enable kubernetes
    2  vault write auth/kubernetes/config token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    3  vault kv put secret/vault-demo-secret1 username="phil" password="notverysecure"
    4  vault kv get secret secret/vault-demo-secret1
    5  vault kv get secret/vault-demo-secret1
    6  history
    7  exit
    8  vault write auth/kubernetes/config kubernetes_host=https://172.30.0.1:443 disable_iss_validation=false
    9  OCP_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
   10  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": ""}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   11  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "pmodemo"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   12  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "pmodemo"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   13  vault write auth/kubernetes/config kubernetes_host=https://172.30.0.1:443 disable_iss_validation=true
   14  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "pmodemo"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   15  clear
   16  exit
   17  curl     --header "X-Vault-Token: ..."     http://127.0.0.1:8200/v1/auth/kubernetes/config
   18  OCP_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
   19  curl     --header "X-Vault-Token: ..."     http://127.0.0.1:8200/v1/auth/kubernetes/config
   20  vault write auth/kubernetes/config token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
   21  vault read auth/kubenertes/config
   22  vault read auth/kubernetes/config
   23  OCP_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
   24  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "pmodemo"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   25  curl -k --request POST --data '{"jwt": "$OCP_TOKEN", "role": "pmodemo"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   26  curl -k --request POST --data '{"jwt": '$OCP_TOKEN', "role": "pmodemo"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   27  curl -k --request POST --data '{"jwt": $OCP_TOKEN, "role": "pmodemo"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   28  curl -k --request POST --data '{"jwt": `$(cat /var/run/secrets/kubernetes.io/serviceaccount/token`, "role": "pmodemo"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   29  curl -k --request POST --data '{"role": "pmodemo"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   30  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "vplugin"}' http://vault.vault.svc:8200/v1/auth/kubernetes/login
   31  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "vplugin"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   32  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "pmodemo"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   33  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "pmodemo"}' http://127.0.0.1:8200/v1/auth/kubernetes/login
   34  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "pmodemo"}' http://kubernetes.default.svc:8200/v1/auth/kubernetes/login
   35  exit
   36  curl --silent http://127.0.0.1:8001/.well-known/openid-configuration | jq -r .issuer
   37  curl --silent http://127.0.0.1:8001/.well-known/openid-configuration 
   38  curl --silent http://127.0.0.1:8000/.well-known/openid-configuration 
   39  vault write auth/kubernetes/config token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt issuer=https://kubernetes.default.svc
   40  vault read auth/kubernetes/config
   41  curl https://kubernetes.default.svc
   42  curl -k https://kubernetes.default.svc
   43  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "pmodemo"}' http://kubernetes.default.svc:8200/v1/auth/kubernetes/login
   44  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "pmodemo"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   45  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "pmodemo"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   46  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "pmodemo"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   47  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "vplugin"}' http://vault.vault.svc:8200/v1/auth/kubernetes/login
   48  history | grep POST
   49  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "vplugin"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   50  history | grep export
   51  OCP_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
   52  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "vplugin"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   53  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "pmodemo"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   54  history
   55  history | grep read
   56  vault read auth/kubernetes/config
   57  vault read auth/kubernetes/config
   58  curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "pmodemo"}' http://172.30.46.242:8200/v1/auth/kubernetes/login
   59  history
                                          
                                          
 curl --silent http://127.0.0.1:8001/api/v1/namespaces/default/serviceaccounts/default/token   -H "Content-Type: application/json"   -X POST   -d '{"apiVersion": "authentication.k8s.io/v1", "kind": "TokenRequest"}'   | jq -r '.status.token'   | cut -d. -f2   | base64 -d
{"aud":["https://kubernetes.default.svc"],"exp":1631646050,"iat":1631642450,"iss":"https://kubernetes.default.svc","kubernetes.io":{"namespace":"default","serviceaccount":{"name":"default","uid":"466621df-acac-43f6-ac8d-de56e04213a3"}},"nbf":1631642450,"sub":"system:serviceaccount:default:default"}base64: invalid input

    kubectl proxy &
curl --silent http://127.0.0.1:8001/.well-known/openid-configuration | jq -r .issuer

                                          
                                          
   ****vault server -config=/etc/vault/config-file.hcl -log-level=debug*****
                                          
                                          
 to view / enable logging = ****vault audit enable file file_path=/var/log/vault_audit.log****
                                         **** vault audit enable syslog ****
