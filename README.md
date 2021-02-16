# Vault & consul Kubernetes Deployment
Following this project, you will be able to deploy, configure and use an [HashiCorp Vault](https://www.vaultproject.io/) with [HashCorp Consul](https://www.consul.io/) and try it in your kubernetes Cluster with sample application.

<img src="vault.png" width="99%">

> Secure, store and tightly control access to tokens, passwords, certificates, encryption keys for protecting secrets and other sensitive data using a UI, CLI, or HTTP API.

<img src="consul.png" width="99%">

> Consul is a service mesh solution providing a full featured control plane with service discovery, configuration, and segmentation functionality. Each of these features can be used individually as needed, or they can be used together to build a full service mesh


## 1- Stack :
* Kubelet : **v1.17.2 / v1.18.5**
* Kubectl : **v1.17.1**
* Docker : **19.03.5 / 19.03.8**
* Consul : **1.9.3**
* Vault : **1.6.2 (Agent 0.8.0)**
* Cfssl : **1.2.0**
* Kube namespace : **vault** *(if you use a different namespace, it must be changed in service and pod hostnames)*
* Architecture : **AMD64 / ARM64**

## 2- Consul deployment :
1. First, generate SSL certificates for Consul (can be done in workstation) with [Cfssl](https://cfssl.org/), after editiing configuration files in [consul/ca](consul/ca) directory

```bash
# Generate CA ans sign request for Consul
cfssl gencert -initca ca/ca-csr.json | cfssljson -bare ca
# Generate SSL certificates for Consul
cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca/ca-config.json \
-profile=default \
ca/consul-csr.json | cfssljson -bare consul
# Perpare a GISSIP key for Consul members communiction encryptation
GOSSIP_ENCRYPTION_KEY=$(consul keygen)
```

2. Create secret with gossip key and public/private keys
```bash
kubectl create secret gesneric consul \
--from-literal="gossip-encryption-key=${GOSSIP_ENCRYPTION_KEY}" \
--from-file=ca.pem \
--from-file=consul.pem \
--from-file=consul-key.pem
```

3. Deploy 3 Consul members (Statefulset)
```bash
kubectl apply -f consul/service.yaml
kubectl apply -f consul/rbac.yaml
kubectl apply -f consul/config.yaml
kubectl apply -f consul/consul.yaml
```

4. Prepare SSL certificates for Consul client, it will be used by vault consul client (sidecar).
```bash
cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca/ca-config.json \
-profile=default \
ca/consul-csr.json | cfssljson -bare client-vault
```

5. Create secret for Consul client (like members)
```bash
kubectl create secret generic client-vault \
--from-literal="gossip-encryption-key=${GOSSIP_ENCRYPTION_KEY}" \
--from-file=ca.pem \
--from-file=client-vault.pem \
--from-file=client-vault-key.pem
```

## 3- Vault deployment :
Before deploy Vaul, e nned to configure a Consul client to give Vault access to Consul members
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: vault-config
data:
  ...
  concul.config: |
    {
      "verify_incoming": false,
      "verify_outgoing": true,
      "server": false,
      "ca_file": "/etc/tls/ca.pem",
      "cert_file": "/etc/tls/client-vault.pem",
      "datacenter": "vault",
      "key_file": "/etc/tls/client-vault-key.pem",
      "client_addr": "127.0.0.1",
      "ui": false,
      "raft_protocol": 3,
      "retry_join": [ "provider=k8s label_selector=\"app=consul,role=server\" namespace=\"vault\"" ]
    }
```
The consul client will be deployed as a Sidecar for Vault server, so the ***"client_addr"*** must be ***"127.0.0.1"***. For certficates parameters, we will use the *client-vault* secret and same join expression of memberes configuration.

In another side, we need to configure Vault to request this client by the ***"listeners"*** parameter.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: vault-config
data:
  vault.config: |
    {
      "ui": true,
      "listener": [{
        "tcp": {
          "address": "0.0.0.0:8200",
          "tls_disable": true
        }
      }],
...
```
OK, let's deploy
```bash
kubectl apply -f vault/service.yaml
kubectl apply -f vault/config.yaml
kubectl apply -f vault/vault.yaml
```

## 5- UI:
A this point, we have 3 instance on Consul deployed and 1 instance of Vault connected to Consul members.

We can use a port forwarding to acces UI of Consul and Vault. In this case, we will to use an ***"Ingress"*** to expose UIs to internet.

```bash
kubectl apply -f ingress.yaml
```
> If you use this option with SSL (HTTPS), you need to configure the TLS secret.

## 6- Vault Injector deployment
* Install vault agent injector (single and simple instance without leader & leader election)
```bash
kubectl apply -f vault-injector/serivce.yaml
kubectl apply -f vault-injector/rbac.yaml
kubectl apply -f vault-injector/deployment.yaml
kubectl apply -f vault-injector/webhook.yaml # webhook must be created after deployment
```

> The injector will dected Vault an ***"Annotation"*** or a ***"Configmap"***, and will inject an ***initContainer*** in init process of your application Pod to request vault for secret. After initialisation, an agent will be injected inside the pod to give your application container the requested secret.

> For agent injector, we use own [docker image](https://hub.docker.com/repository/docker/medinvention/kubernetes-vault), it's similaire of offical image but support arm64 (At 02/2021 only amd64 arch are distributed by hashicorp), see [docker files](docker/).

## 7- Sample deployment :
We can use UI to configure and use Vault, in this project we use CLI.
1. Start by installing vault locally (in workspace) for CLI use only
```bash
curl https://releases.hashicorp.com/vault/1.6.2/vault_1.6.2_linux_amd64.zip -o vault_1.6.2_linux_amd64.zip
unzip vault_1.6.2_linux_amd64.zip
chmod +x vault
# With ingress, you can use the root url of vault ui, or use the port forward
export VAULT_ADDR="YOUR_VAULT_HOST"
```

2. Check the server status and login (using token like UI)
```bash
./vault status
>> Key             Value
>> ---             -----
>> Seal Type       shamir
>> Initialized     true
>> Sealed          false
>> Total Shares    1
>> Threshold       1
>> Version         1.6.2
>> Storage Type    consul
>> Cluster Name    vault-cluster-...
>> Cluster ID      ...
>> HA Enabled      true
>> HA Cluster      ..
>> HA Mode         active

./vault login
<< your_token
```

3. Create key/value for testing
```bash
./vault secrets enable kv
./vault kv put kv/myapp/config username="admin" password="adminpassword"
```

4. Connect kube to vault
```bash
# Create the service account to access secret
kubectl apply -f myapp/service-account.yaml
# Enable kubernetes support
./vault auth enable kubernetes
# Prepare kube api server data
export SECRET_NAME="$(kubectl get serviceaccount vault-auth  -o go-template='{{ (index .secrets 0).name }}')"
export TR_ACCOUNT_TOKEN="$(kubectl get secret ${SECRET_NAME} -o go-template='{{ .data.token }}' | base64 --decode)"
export K8S_API_SERVER="$(kubectl config view --raw -o go-template="{{ range .clusters }}{{ index .cluster \"server\" }}{{ end }}")"
export K8S_CACERT="$(kubectl config view --raw -o go-template="{{ range .clusters }}{{ index .cluster \"certificate-authority-data\" }}{{ end }}" | base64 --decode)"
# Send kube config to vault
./vault write auth/kubernetes/config kubernetes_host="${K8S_API_SERVER}" kubernetes_ca_cert="${K8S_CACERT}" token_reviewer_jwt="${TR_ACCOUNT_TOKEN}"
```

5. Create Vault policy and role for "myapp"

Edit policy file myapp/policy.json
```yaml
path "kv/myapp/*" {
  capabilities = ["read", "list"]
}
```
Create the application role
```bash
./vault policy write myapp-ro myapp/policy.json
./vault write auth/kubernetes/role/myapp-role bound_service_account_names=vault-auth bound_service_account_namespaces=vault policies=default,myapp-ro ttl=15m
```

6. Deploy "myapp" for testing

Edit annotations for secret output, [@see myapp/deployment.yaml](myapp/deployment.yaml)
```yaml
annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/agent-inject-secret-account: "kv/myapp/config"
    vault.hashicorp.com/agent-inject-template-account: |
        {{- with secret "kv/myapp/config" -}}
        dsn://{{ .Data.username }}:{{ .Data.password }}@database:port/mydb?sslmode=disable
        {{- end }}
    vault.hashicorp.com/role: "myapp-role"
```
Deploy app and log secret
```bash
kubectl apply -f myapp/deployment.yaml 
export POD=$(kubectl get pods --selector=app=myapp --output=jsonpath={.items..metadata.name})
kubectl log ${POD} myapp
```


## 8- Tips
To activate an HTTP Basic security for Consul UI (it's run without), you can use Nginx ingress annotations,
after authentication secret generation.
```bash
htpasswd -c auth foo
kubectl create secret generic consul-auth --from-file=auth
```
Add annotations
```yaml
nginx.ingress.kubernetes.io/auth-type: basic
nginx.ingress.kubernetes.io/auth-secret: consul-auth
nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - Consul - MedInvention'
```


## Links
- https://github.com/kelseyhightower/consul-on-kubernetes
- https://github.com/hashicorp/vault-k8s
- https://www.hashicorp.com/blog/whats-next-for-vault-and-kubernetes
- https://github.com/hashicorp/vault-helm
- https://medium.com/hashicorp-engineering/hashicorp-vault-delivering-secrets-with-kubernetes-1b358c03b2a3
- https://github.com/hashicorp/consul
- https://blog.zwindler.fr/2020/08/31/gerez-vos-secrets-kubernetes-dans-vault/

---- 

[*More information*](https://blog.medinvention.dev)