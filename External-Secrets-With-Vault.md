**Background**

I’ve been spending a fair amount of time researching secrets management with OpenShift. The interest started with the IBM Vault plugin for ArgoCD, which allows us to store placeholders for secrets in Git but when used with the OpenShift GitOps operator it requires a fair amount of configuration and maintenance. I asked other co-workers to join in a roundtable to discuss secrets management and surrounding tooling. Among the leading interest in this space was “External Secrets” and so the journey began to start implementing each tool and comparing these not only at a platform level but also with consideration for GitOps and ArgoCD. At no point is a single solution being proposed as the best path forward for every case, development and operation requirements must be considered with each tool’s pros and cons in mind. This proof of concept is based on a Hashicorp Vault back end, as I have utilized this with several customers recently. 


**What are External-Secrets?**

External-Secrets extends the Kubernetes API vi an ExternalSecrets object + a controller. In short, the ExternalSecret object declares how/where to fetch the secret data and in turn the controller converts that to a secret in the namespace. In the case of GitOps, utilizing External-Secrets allows you to store the External-Secret in Git without exposing a secret in Git or the tooling (Argo, Flux, etc). In the case of application consumption of secrets, pods are able to utilize secrets just as they normally would, with little maintenance or overhead, the External-Secrets controller creates the secret based on the eternal-secret manifest. Everbody knows the rules... NO SECRETS IN GIT! utilizing External-Secrets allows us to play by the rules. 

**Assumptions for this Demo**

- Access to a working OpenShift cluster. If a Cluster is not available, Code Ready Containers can be utilized to PoC. https://developers.redhat.com/products/codeready-containers/overview
- A Hashicorp vault implementation. In the demo, dev mode is utilized with a single pod. This is not meant for production and should not be run outside of sandbox or development environment
- A secret to store. 


**it is important to note that there has been concern around the default helm chart used for deploying the dev Vault environment. The steps taken here are a proof of concept and the dev Vault should not be run with the default config outside of a sandbox/dev environment.**


Utilizing External-Secrets is an incredibly simple process consisting of installing the tooling and creating  your external-secret manifest based on secrets management back end in use. Secrets management back ends are not limited to Vault, you can see all of the documentation on External=Secrets here: https://github.com/external-secrets/kubernetes-external-secrets

A large portion of this demo will revolve around configuring Vault. We will touch on the concepts but not deep dive into the advanced configuration options of Vault. 

Assuming an OpenShift cluster or CRC is installed and you are logged in, we can create 2 projects, one for external secrets and one for the dev vault. We will also add the Helm repositories for each of the two tools we will use. 

**Lets Create the projects and add the Helm repositories**

``oc new-project vault``

``helm repo add hashicorp https://helm.releases.hashicorp.com``

``oc new-project external secrets`` 

``helm repo add external-secrets https://external-secrets.github.io/kubernetes-external-secrets/``



**Configuring Vault**

When utilizing Vault as a secrets manager back end to store secrets we can consider the steps below for a working implementation. Sealing and unsealing the Vault is out of scope for the demo, the Vault will be unsealed when installed. We are installing Vault from a Helm chart, no injector is being installed. I’m approaching this to concentrate on the strengths of External-Secrets and the ability to allow the tooling to handle secrets without additional injection. Only a single vault-0 pod will start as an ephemeral Vault.

- Enable Kubernetes authentication.
- Configure authentication to utilize the pod service account token and cert of the k8s host. 
- Create a secret.
- Create a Vault policy to access the secret (or secret path.)
- Create a role to associate with the policy created earlier. 

``oc project vault`` 

``helm install vault hashicorp/vault --set "global.openshift=true" --set "server.dev.enabled=true" --set="injector.enabled=false"``

```NAME: vault
LAST DEPLOYED: Thu Sep 16 12:10:00 2021
NAMESPACE: vault
STATUS: deployed
REVISION: 1
NOTES:
Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using Vault with Kubernetes available here:https://www.vaultproject.io/docs/
```
**Image Tag Caution**
(bonus exercise!)

I have witnessed an outdated tag being used in the Helm chart. If you are seeing an image pull error, investigate the image within your Vault statefulset and adjust your chart or deployment accordingly. For a quick and dirty fix, edit the statefulset and add the image tag of ‘latest’. (This is a quick proof of concept, this will get you up and running to demo the theory.) 

**Configuration**
We begin our journey with Vault by first enabling an authentication method and later, we will configure namespace and service account access. Because we are using a Dev environment, we will configure these at the pod level, utilizing `oc rsh` to access our Vault pod.


Step 0. Rsh to the Vault pod

``oc rsh vault-0``

Step 1. Enable Kubernetes Auth (from within the pod)

``vault auth enable kubernetes``

Step 2. Configure our authentication to utilize the service account token and certificate of the Kubernetes host.

``vault write auth/kubernetes/config token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt issuer=https://kubernetes.default.svc ``

Step 3. We will create a key:value pair password to store in our Vault. 

``vault kv put secret/vault-demo-secret1 username="phil" password="notverysecure"``

Step 4. We create a policy, an hcl or JSON file, which defines the access allowed to the secrets path.

```
vault policy write pmodemo - << EOF
path "secret/data/vault-demo-secret1"
  { capabilities = ["read"]
}
EOF
```


Step 5. We create a role, which defines a namespace and service account with the policy which was created earlier. We are making two, the external-secrets role is required, the vault role is used for testing later. It is important to note that you will need to set the bound_service_account_names and the service_account_namespaces to those which are present in the deployment (external-secrets) or statefulset (vault) for your cluster. 

``vault write auth/kubernetes/role/pmodemo1 bound_service_account_names=vault bound_service_account_namespaces=vault policies=pmodemo ttl=60m``

``vault write auth/kubernetes/role/pmodemo bound_service_account_names=external-secrets-kubernetes-external-secrets bound_service_account_namespaces=external-secrets policies=pmodemo ttl=60m``

At this point, we've enabled Kubernetes authentication, configured our auth, created a secret, policy and role and we should now be able to interact with the Vault API. Let's test this:

From the pod that will be accessing Vault, export the token from the ServiceAccount which the pod runs as. It's important to note here, if your pod is not mounting a service account token, i.e.   automountServiceAccountToken: false is set in the pod spec, you will not be able to utilize the kubernetes authentication method.

```oc rsh vault-0
OCP_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

Now, lets do a Curl request to the Vault host (`oc get svc` in the project will allow you to get the IP:port of your Vault)

```curl -k --request POST --data '{"jwt": "'"$OCP_TOKEN"'", "role": "pmodemo1"}' http://10.217.5.249:8200/v1/auth/kubernetes/login```

{"request_id":"5e833fc7-4f53-f7dc-edfe-2257a42793d1","lease_id":"","renewable":false,"lease_duration":0,"data":null,"wrap_info":null,"warnings":null,"auth":{"client_token":"s.ojvma4OkSZT7qKKRj7qYDswv","accessor":"K0pI3iNha8dUi3YCAxAzumRB","policies":["default","pmodemo"],"token_policies":["default","pmodemo"],"metadata":{"role":"pmodemo","service_account_name":"vault","service_account_namespace":"vault","service_account_secret_name":"","service_account_uid":"9218e2d0-56dd-48b6-b544-d136f79297a2"},"lease_duration":3600,"renewable":true,"entity_id":"05e3113e-7685-137a-c2c4-42fafcf5f71a","token_type":"service","orphan":true}


If you see something similar to the above, Vault is installed and authentication is working. Now it is time to install and create an external secret! If you see an error, investigate the error appropriately, most common errors:
- Error: connect EHOSTUNREACH = the vault endpoint env var in external secrets deployment is incorrect
- ERROR, namespace not authorized = the namespace is not set properly in the role 
- ERROR, service account name not authorized = the service account is not proper in the role
- 403 permission denied = review your policy

**External-Secrets Deployment and Configuration**

``oc project external-secrets``

``helm install external-secrets external-secrets/kubernetes-external-secrets``

The deployment of External-Secrets relies on environment variables to configure where/how to reach the Vault API. You can set the VAULT_ADDR variable to the IP:Port of your Vault implementation:
        - name: VAULT_ADDR
          value: http://10.217.5.249:8200 ``

Lets now create the external-secret manifest. We will need to consider the data for the secret (being extracted from Vault) as well as the vaultMountPoint and vaultRole. 

```
apiVersion: kubernetes-client.io/v1
kind: ExternalSecret
metadata:
  name: exsecret1
  namespace: vault 
spec:
  backendType: vault
  data:
    - key: secret/data/vault-demo-secret1
      name: password
      property: password
  vaultMountPoint: kubernetes
  vaultRole: pmodemo
```

``oc create -f extsecret1.yml``

**Check the Results**

When we successfully create the external-secret, we should in turn see a secret created on the cluster. So order of operations: external secret created, data pulled from Vault, ES controller creates cluster level secret. Only the Vault secret and the cluster secret should have the actual secret data, the external secret will contain the placeholder data. 

```
oc get es

NAME            LAST SYNC   STATUS    AGE
exsecret1       6s          SUCCESS   22h

```

Finally, we take a look at the secret which was in turn created by the external-secrets controller as well as the data in the ES. We see password data in the secret but not in the external-secret, this allows us to store the ES in Git without ever exposing the secret.  

``oc get secrets``

``` 
NAME                          TYPE                                  DATA   AGE
exsecret1                     Opaque                                1      2m29s
```


``` 
oc get secret exsecret1 -o yaml
apiVersion: v1
data:
  password: bm90dmVyeXNlY3VyZQ==
kind: Secret
….
```
You can decide the secret data and compare it to the secret you setup earlier:
```
echo "bm90dmVyeXNlY3VyZQ==" | base64 -d 
notverysecure
``` 

``oc get es exsecret1 -o yaml``

And we verify there is no data in the manifest which would present a risk

```
spec:
  backendType: vault
  data:
  - key: secret/data/vault-demo-secret1
    name: password
    property: password
  vaultMountPoint: kubernetes
  vaultRole: pmodemo
```

It’s just that simple. In this demo we've setup a dev Vault, enabled Kubernetes authentication, created a secret, a role and policy for the secret access. 





