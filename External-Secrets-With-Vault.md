# External Secrets with Hashicorp Vault

## Background

I’ve been spending a fair amount of time researching secrets management with OpenShift. The interest started with the [IBM Vault Plugin](https://github.com/IBM/argocd-vault-plugin) for [Argo CD](https://argoproj.github.io/argo-cd/), which allows us to store placeholders for [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) in Git. But, when used with the OpenShift GitOps operator, a fair amount of configuration and maintenance is required. I asked other co-workers to join in a roundtable to discuss secrets management and surrounding tooling. Among the leading interest in this space was [Kubernetes External Secrets](https://github.com/external-secrets/kubernetes-external-secrets) (External Secrets) and so the journey began to start implementing each tool and comparing these not only at a platform level, but also with consideration for GitOps and Argo CD. At no point is a single solution being proposed as the best path forward for every use case. Development and operation requirements must be considered along with the pros and cons of each tool in mind. This proof of concept is based on a [Hashicorp Vault](https://www.vaultproject.io/) back end, as I have utilized this tool with several customers recently. 


## What are External Secrets?

External Secrets extends the Kubernetes API vi an _ExternalSecrets_ object + a controller. In short, the _ExternalSecret_ object declares how and where to fetch the secret data from the external source, and in turn, the controller converts that resource into a secret in the namespace for which the ExternalSecret is created. In the case of GitOps, utilizing external-secrets allows you to store the _ExternalSecret_ in Git without exposing the sensitive asset in Git or in the GitOps tool (Argo, Flux, etc). In the case of application consumption of secrets, pods are still able to utilize secrets just as they normally would. The External Secrets controller creates the secret based on the _ExternalSecrets_ manifest. Everybody knows the rules... NO SECRETS IN GIT! Utilizing External Secrets allows us to abide by by these rules. 

## Assumptions for this Demo

- Access to a working OpenShift cluster. If a Cluster is not available, [Code Ready Containers](https://developers.redhat.com/products/codeready-containers/overview) can be utilized for this exercise.
- A Hashicorp Vault implementation. In the demo, [dev mode](https://www.vaultproject.io/docs/concepts/dev-server) is utilized which deploys a single pod instance. "Dev mode" is not meant for production and should not be run outside of sandbox or development environment
- A secret to store. 


Deploying External Secrets is an incredibly simple process consisting of installing the tooling and creating your _ExternalSecret_ manifest based on secrets management back end in use. Secrets management backends are not limited to Hashicorp Vault as External Secrets supports a number of providers. A full list of supported backends can be found [here](https://github.com/external-secrets/kubernetes-external-secrets#backends)

A large portion of this demo will revolve around configuring Vault. We will touch on the basic concepts, but not dive into the advanced configuration options available. 

Assuming an OpenShift cluster or CRC is available and you are currently logged in, create 2 projects: one for external secrets and one for the dev vault instance. We will also add [Helm](https://helm.sh) repositories for each of the two tools. 

## Lets Create the projects and add the Helm repositories

```shell
oc new-project external-secrets
```

```shell
helm repo add external-secrets https://external-secrets.github.io/kubernetes-external-secrets/
```

```shell
oc new-project vault
```

```shell
helm repo add hashicorp https://helm.releases.hashicorp.com
```

## Configuring Vault

When utilizing Vault as a secrets manager back end to store secrets, we can consider the steps below for a working implementation. [Sealing and unsealing](https://www.vaultproject.io/docs/concepts/seal) the Vault is out of scope for the demo as the Vault will be unsealed when installed using "Dev Mode". We are installing Vault from a Helm chart without the use of the [Vault Agent Injector](https://www.vaultproject.io/docs/platform/k8s/injector). I’m approaching this to concentrate on the strengths of External Secrets and the ability to allow the tooling to handle secrets without additional injection. Only a single `vault-0` pod will start as an ephemeral Vault instance.

Change into the `vault` project and deploy Vault using Helm.

```shell
oc project vault
``` 

```shell
helm upgrade -i -n vault vault hashicorp/vault --set "global.openshift=true" --set "server.dev.enabled=true" --set="injector.enabled=false" --set="server.image.repository=docker.io/hashicorp/vault"
```

```shell
NAME: vault
LAST DEPLOYED: Thu Sep 16 12:10:00 2021
NAMESPACE: vault
STATUS: deployed
REVISION: 1
NOTES:
Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using Vault with Kubernetes available here:https://www.vaultproject.io/docs/
```

## Configuration

We begin our journey with Vault by first enabling an authentication method and later, we will configure namespace and service account access. Because we are using a Dev environment, we will implement these configurations at the pod level utilizing `oc rsh` to access our Vault pod.

In particular, following steps will be performed: 

- Enable [Kubernetes authentication](https://www.vaultproject.io/docs/auth/kubernetes).
- Configure authentication to utilize the pod service account token and cert of the k8s host. 
- Create a secret.
- Create a [Vault Policy](https://www.vaultproject.io/docs/concepts/policies) to access the secret (or secret path.)
- Create a [role](https://www.vaultproject.io/docs/auth/approle) to associate with the policy created earlier. 


Step 0. Execute a remote shell session (rsh) to the Vault pod

```shell
oc rsh vault-0
```

Step 1. Enable Kubernetes Auth (from within the pod)

```shell
vault auth enable kubernetes
```

Step 2. Configure our authentication to utilize the service account token mounted in the pod and certificate of the Kubernetes cluster.

```shell
vault write auth/kubernetes/config token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt issuer=https://kubernetes.default.svc
```

Step 3. We will utilize the [KV secrets engine](https://www.vaultproject.io/docs/secrets/kv) and create a key:value pair password to store in our Vault. 

```shell
vault kv put secret/vault-demo-secret1 username="phil" password="notverysecure"
```

Step 4. Create a policy (an hcl or JSON file) which defines the access allowed to the secrets path.

```shell
vault policy write pmodemo - << EOF
path "secret/data/vault-demo-secret1"
  { capabilities = ["read"]
}
EOF
```

Step 5. Create roles to associate the namespace and service account with the policy which was created earlier. Two roles will be created: one for the external secrets namespace and the other for testing later on. It is important to note that you will need to set the `bound_service_account_names` and the `service_account_namespaces` to those associated with the Deployment (`external-secrets`) or StatefulSet (`vault`). 

```shell
vault write auth/kubernetes/role/pmodemo1 bound_service_account_names=vault bound_service_account_namespaces=vault policies=pmodemo ttl=60m
```

```shell
vault write auth/kubernetes/role/pmodemo bound_service_account_names=external-secrets-kubernetes-external-secrets bound_service_account_namespaces=external-secrets policies=pmodemo ttl=60m
```

At this point, we've enabled Kubernetes authentication, configured our auth, created a secret, policy, and role, and we should now be able to interact with the Vault API. Let's test this:

From the pod that will be accessing Vault, export the Service Account token. It is important to note here that if your pod is not mounting a service account token (`automountServiceAccountToken: false`), you will not be able to utilize the Kubernetes Auth method.

```shell
OCP_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

Now, lets make a request to Vault to validate out setup:

```shell
wget --no-check-certificate -q -O- --post-data '{"jwt": "'"$OCP_TOKEN"'", "role": "pmodemo1"}' http://vault:8200/v1/auth/kubernetes/login
```

{"request_id":"5e833fc7-4f53-f7dc-edfe-2257a42793d1","lease_id":"","renewable":false,"lease_duration":0,"data":null,"wrap_info":null,"warnings":null,"auth":{"client_token":"s.ojvma4OkSZT7qKKRj7qYDswv","accessor":"K0pI3iNha8dUi3YCAxAzumRB","policies":["default","pmodemo"],"token_policies":["default","pmodemo"],"metadata":{"role":"pmodemo","service_account_name":"vault","service_account_namespace":"vault","service_account_secret_name":"","service_account_uid":"9218e2d0-56dd-48b6-b544-d136f79297a2"},"lease_duration":3600,"renewable":true,"entity_id":"05e3113e-7685-137a-c2c4-42fafcf5f71a","token_type":"service","orphan":true}


If a response similar to the above is displayed, Vault is installed and authentication is working. Now it is time to install and create an external secret! If you see an error, investigate the error appropriately. One of the most common errors is as follows:

```shell
- Error: connect EHOSTUNREACH = the vault endpoint env var in external secrets deployment is incorrect
- ERROR, namespace not authorized = the namespace is not set properly in the role 
- ERROR, service account name not authorized = the service account is not proper in the role
- 403 permission denied = review your policy
```

## External Secrets Deployment and Configuration**

Next, install and configure External Secrets:

```shell
oc project external-secrets
```

```shell
helm upgrade -i -n external-secrets external-secrets external-secrets/kubernetes-external-secrets --set "env.VAULT_ADDR=http://vault.vault.svc:8200"
```

The deployment of External Secrets relies on environment variables to configure where/how to reach the Vault API. This is set via the Helm value passed into the command above referencing the location of the Vault instance.

Let's now create the _ExternalSecret_ manifest in a file called `extsecret1.yml` to reference the secret created in Vault previously. We will need to specify the `vaultMountPoint` and `vaultRole` properties to refer to the location of the secret within Vault. 

```shell
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

```shell
oc create -f extsecret1.yml
```

## Check the Results

When we successfully create the _ExternalSecret_ manifest, the External Secrets controller will create a Kubernetes Secret on the cluster containing the secret stored in Vault. So order of operations: 

1. _ExternalSecret_ created
2. Data pulled from Vault
3. External Secret controller creates Kubernetes secret.

Only the Vault secret and the cluster secret should have the actual secret data. The _ExternalSecret_ will contain just the reference. 

```
oc get es -n vault

NAME            LAST SYNC   STATUS    AGE
exsecret1       6s          SUCCESS   22h

```

Finally, we can take a look at the secret that was created by the External Secrets controller as well as the data in the _ExternalSecret_. We see password data in the secret but not in the _ExternalSecret_, which allows us to store the _ExternalSecret_ in Git without ever exposing the actual secret data.  

```shell
oc -n vault get secrets exsecret1
```

```shell 
NAME                          TYPE                                  DATA   AGE
exsecret1                     Opaque                                1      2m29s
```

```shell
oc -n vault get secret exsecret1 -o yaml
apiVersion: v1
data:
  password: bm90dmVyeXNlY3VyZQ==
kind: Secret
….
```

You can view the decoded secret data and compare it to the secret you setup earlier:

```shell
oc -n vault extract secret/exsecret1 --to=-
# password
notverysecure
``` 

And we can once again verify that there is no sensitive data in the _ExternalSecret_ manifest which would present a risk when committed to a git repository:

```yaml
spec:
  backendType: vault
  data:
  - key: secret/data/vault-demo-secret1
    name: password
    property: password
  vaultMountPoint: kubernetes
  vaultRole: pmodemo
```

It’s just that simple. In this demo, we setup an instance of Vault in "dev mode", enabled Kubernetes authentication, created a secret, a role and policy to manage access to the secret. We then created an _ExternalSecret_ which holds the path to the sensitive data in Vault and allowed the controller to create the secret in OpenShift. One final reminder, in closing. Secrets are stored as an encoded value in etcd. To help protect secrets at rest, pursue encrypting etcd: [https://docs.openshift.com/container-platform/4.7/security/encrypting-etcd.html](https://docs.openshift.com/container-platform/4.7/security/encrypting-etcd.html).





