# k8s-hashicorp-vault-agent-nginx-web

1. Deploy a local development Vault Server inside Kubernetes
2. Configure Vault for PKI certificate management (self-signed)
3. Use Vault Agent to write certificates to a shared location to be used by NGINX as SSL/TLS certificate.

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

Start an interactive shell session on the `vault-0` pod.

```shell
kubectl exec -it vault-0 -- /bin/sh
```
Enable the pki secrets engine at the pki path.

```shell
vault secrets enable pki
```

Tune the pki secrets engine to issue certificates with a maximum time-to-live (TTL) of 87600 hours.

```shell
vault secrets tune -max-lease-ttl=87600h pki
```

Generate the example.com root CA, give it an issuer name, and save its certificate in the file root_2024_ca.crt

```shell
vault write -field=certificate pki/root/generate/internal \
     common_name="example.com" \
     issuer_name="root-2024" \
     ttl=87600h > root_2024_ca.crt
```

Create a role for the root CA. Creating this role allows for specifying an issuer when necessary for the purposes of this scenario. This also provides a simple way to transition from one issuer to another by referring to it by name.

```shell
vault write pki/roles/2024-servers allow_any_name=true
```

Configure the CA and CRL URLs.

```shell
vault write pki/config/urls \
     issuing_certificates="$VAULT_ADDR/v1/pki/ca" \
     crl_distribution_points="$VAULT_ADDR/v1/pki/crl"
```

Enable the Kubernetes auth method:

```shell
vault auth enable kubernetes
```

Configure the Kubernetes authentication method to use the location of the Kubernetes API.

```shell
vault write auth/kubernetes/config \
      kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
```

Write out the policy named internal-app that enables the read and create capability for secrets at path pki/issue/2024-servers.

```shell
vault policy write internal-app - <<EOF
path "pki/issue/2024-servers" {
   capabilities = ["read", "create"]
}
EOF
```

Create a Kubernetes authentication role named internal-app.

```shell
vault write auth/kubernetes/role/internal-app \
      bound_service_account_names=internal-app \
      bound_service_account_namespaces=default \
      policies=internal-app \
      ttl=24h
```

Exit the vault-0 pod.

```shell
exit
```

Create a Kubernetes service account named internal-app in the default namespace.

```shell
kubectl create sa internal-app
```

Create configmap for nginx default.conf file

```shell
kubectl create configmap nginxconfigmap --from-file=default.conf
```

Apply the deployment & service defined in nginx-vault.yaml

```shell
kubectl apply -f nginx-vault.yaml
```
