# vault-SPIRE-OIDC-K8s-Workloads
This guide is take from [Spiffe Using SPIRE and OIDC to Authenticate Workloads to Retrieve Vault Secrets](https://spiffe.io/docs/latest/keyless/vault/readme/)
This tutorial builds on the Kubernetes Quickstart guide to describe how to set up OIDC Federation between a SPIRE Server and a Vault server. This will allow a SPIRE-identified workload to authenticate against a federated Vault server by presenting no more than its JWT-SVID. Using this technique the workload won’t need to authenticate itself against the Vault server using another authentication method like AppRole or Username & Password.

In this tutorial you will learn how to:

* Deploy the OIDC Discovery Provider Service
* Create the required DNS A record to point to the OIDC Discovery document endpoint
* Set up a local Vault server to store secrets
* Configure a SPIRE Server OIDC provider as an authentication method for the Vault server
* Test access to secrets using a SPIRE-provided identity
---

## Deploy Spiffe
```bash
cd spire-tutorials/k8s/quickstart 

kubectl apply -f spire-namespace.yaml

kubectl apply \
    -f server-account.yaml \
    -f spire-bundle-configmap.yaml \
    -f server-cluster-role.yaml

kubectl apply \
    -f server-configmap.yaml \
    -f server-statefulset.yaml \
    -f server-service.yaml

kubectl get statefulset --namespace spire

kubectl apply \
    -f agent-account.yaml \
    -f agent-cluster-role.yaml

kubectl apply \
    -f agent-configmap.yaml \
    -f agent-daemonset.yaml

kubectl exec -n spire spire-server-0 -- \
    /opt/spire/bin/spire-server entry create \
    -spiffeID spiffe://example.org/ns/spire/sa/spire-agent \
    -selector k8s_sat:cluster:demo-cluster \
    -selector k8s_sat:agent_ns:spire \
    -selector k8s_sat:agent_sa:spire-agent \
    -node

kubectl exec -n spire spire-server-0 -- \
    /opt/spire/bin/spire-server entry create \
    -spiffeID spiffe://example.org/ns/default/sa/default \
    -parentID spiffe://example.org/ns/spire/sa/spire-agent \
    -selector k8s:ns:default \
    -selector k8s:sa:default

kubectl apply -f client-deployment.yaml

kubectl exec -it $(kubectl get pods -o=jsonpath='{.items[0].metadata.name}' \
   -l app=client)  -- /opt/spire/bin/spire-agent api fetch -socketPath /run/spire/sockets/agent.sock

cd ../../..
```

## Deploy the OIDC Discovery Provider Configmap

```bash
cd spire-tutorials/k8s/oidc-vault/k8s

kubectl apply \
    -f server-configmap.yaml \
    -f oidc-dp-configmap.yaml \
    -f server-statefulset.yaml

kubectl get pods -n spire -l app=spire-server -o \
    jsonpath='{.items[*].spec.containers[*].name}{"\n"}'

cd ../../../..
```

## Configure the OIDC Discovery Provider Service and Ingress
> **_NOTE:_**  Update domain to `localhost` and email to your email in `spire-tutorials/k8s/oidc-vault/k8s/oidc-dp-configmap.yaml`

> **_NOTE:_**  Update `MY_DISCOVERY_DOMAIN` to `localhost` in `spire-tutorials/k8s/oidc-vault/k8s/ingress.yaml`

```bash

kubectl apply \
    -f server-oidc-service.yaml \
    -f ingress.yaml 
```

## Setup Port Forward
> **_NOTE:_** Do this in another terminal
```bash
kubectl -n spire port-forward services/spire-oidc 443
```

## Create the Config File and Run the Vault Server
> **_NOTE:_** Do this in another terminal

```bash
cd spire-tutorials/k8s/oidc-vault

vault server -config ./vault/config.hcl
```

## Initialize and Unseal the Vault
> **_NOTE:_** We can go back to our original terminal now
```bash
export VAULT_ADDR=http://127.0.0.1:8200

vault operator init
vault operator init
vault operator init

vault login

vault secrets enable -path=secret kv
vault kv put secret/my-super-secret test=123
```