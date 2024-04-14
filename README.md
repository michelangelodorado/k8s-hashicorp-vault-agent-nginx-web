# k8s-hashicorp-vault-agent-nginx-web

Guide to:

1. Deploy a local development Vault Server
2. Configure Vault for PKI certificate management (self-signed)
3. Use Vault Agent to write certificates to a file for applications/NGINX to use.

Pre-requisites:
1. Kubernetes Cluster
2. Helm




Run Vault on Kubernetes is via Helm chart.
Add the HashiCorp Helm repository.

```shell
helm repo add hashicorp https://helm.releases.hashicorp.com
```

Update all the repositories to ensure helm is aware of the latest versions.

```shell
 helm repo update
```

Install the latest version of the Vault server running in development mode.

```shell
 helm install vault hashicorp/vault --set "server.dev.enabled=true"
```

The Vault pod and Vault Agent Injector pod will be deployed in the default namespace.
The `vault-0` pod runs a Vault server in development mode. The `vault-agent-injector` pod performs the injection based on the annotations present or patched on a deployment.

```shell
kubectl exec -it vault-0 -- /bin/sh
```

```shell
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki
```

```shell
vault write -field=certificate pki/root/generate/internal \
     common_name="example.com" \
     issuer_name="root-2024" \
     ttl=87600h > root_2024_ca.crt
```

```shell
vault write pki/roles/2024-servers allow_any_name=true
```

```shell
vault write pki/config/urls \
     issuing_certificates="$VAULT_ADDR/v1/pki/ca" \
     crl_distribution_points="$VAULT_ADDR/v1/pki/crl"
```

```shell
vault auth enable kubernetes
```

```shell
vault write auth/kubernetes/config \
      kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
```

```shell
vault policy write internal-app - <<EOF
path "pki/issue/2024-servers" {
   capabilities = ["read", "create"]
}
EOF
```

```shell
vault write auth/kubernetes/role/internal-app \
      bound_service_account_names=internal-app \
      bound_service_account_namespaces=default \
      policies=internal-app \
      ttl=24h
```
```shell
exit
```

```shell
kubectl create sa internal-app
```

```shell
kubectl create configmap nginxconfigmap --from-file=default.conf
```

```shell
kubectl apply -f nginx-vault.yaml
```
