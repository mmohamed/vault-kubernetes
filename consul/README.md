
@see https://github.com/kelseyhightower/consul-on-kubernetes

> cfssl gencert -initca ca/ca-csr.json | cfssljson -bare ca

> cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca/ca-config.json \
  -profile=default \
  ca/consul-csr.json | cfssljson -bare consul

> export GOSSIP_ENCRYPTION_KEY="qDOPBEr+/oUVeOFQOnVypxwDaHzLrD+lvjo5vCEBbZ0="

> kubectl create secret generic consul \
  --from-literal="gossip-encryption-key=${GOSSIP_ENCRYPTION_KEY}" \
  --from-file=ca.pem \
  --from-file=consul.pem \
  --from-file=consul-key.pem

> kubectl apply -f .

> cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca/ca-config.json \
  -profile=default \
  ca/consul-csr.json | cfssljson -bare client-vault

> kubectl create secret generic client-vault \
  --from-literal="gossip-encryption-key=${GOSSIP_ENCRYPTION_KEY}" \
  --from-file=ca.pem \
  --from-file=client-vault.pem \
  --from-file=client-vault-key.pem

> htpasswd -c auth foo

> kubectl create secret generic consul-auth --from-file=auth

