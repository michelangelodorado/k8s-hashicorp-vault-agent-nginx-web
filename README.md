# k8s-hashicorp-vault-agent-nginx-web

![image](https://github.com/michelangelodorado/k8s-hashicorp-vault-agent-nginx-web/assets/102953584/f31df94c-38a3-4da0-b729-96cc1b84b59f)

We'll set up Vault and the injector service using Vault Helm chart. Then, we'll deploy an HTTPS web app (using NGINX) to demonstrate how the injector service handles secrets and certificates.

In detail:

1. We will deploy a local development Vault Server inside Kubernetes.
2. We will configure the Vault server for PKI certificate management (self-signed).
3. We will configure the Vault Agent injector to generate and write certificates to a shared location to be used by NGINX for SSL/TLS certificates.

Pre-requisites:
- Kubernetes Cluster
- kubectl, Helm


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

> [!NOTE]
> (Optional) Check the service type of `vault`; if it is set to NodePort/Cluster IP, you can change it to type LoadBalancer so you can access the Vault Web GUI.

```shell
kubectl get svc
```

```shell
kubectl edit svc vault
```

Once you've changed it to LoadBalancer, you can access the Vault Portal via `http://<EXTERNAL-IP>:8200/`

Method: **Token**

Token: **root**

---

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
     ttl=43800h
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

Write out the policy named `internal-app` that enables the read and create capability for secrets at path pki/issue/2024-servers.

```shell
vault policy write internal-app - <<EOF
path "pki/issue/2024-servers" {
   capabilities = ["read", "update"]
}
EOF
```

Create a Kubernetes authentication role named `internal-app`.

```shell
vault write auth/kubernetes/role/internal-app \
      bound_service_account_names=internal-app \
      bound_service_account_namespaces=default \
      policies=internal-app \
      ttl=24h
```

Exit the `vault-0` pod.

```shell
exit
```

Create a Kubernetes service account named `internal-app` in the default namespace.

```shell
kubectl create sa internal-app
```

Create configmap for nginx `default.conf` file

```shell
kubectl create configmap nginxconfigmap --from-file=default.conf
```

Apply the deployment & service defined in `nginx-vault.yaml`

```shell
kubectl apply -f nginx-vault.yaml
```

Check the `EXTERNAL-IP` of the `nginxsvc` service

```shell
kubectl get svc
```

Access the NGINX Web Application using web browser `https://<EXTERNAL-IP>/` and you should see the vault generated certificate is used by the application.

## Notes on Vault Agent Injector Annotations

Check the yaml file `nginx-vault.yaml`, you will see Vault agent annotations that was configured. Below are the descriptions of each annotation.

```
vault.hashicorp.com/agent-image: hashicorp/vault:1.5.0
```
Name of the Vault docker image to use. This value overrides the default image configured in the injector and is usually not needed. Defaults to **hashicorp/vault:1.16.1**. I used version 1.5.0 for my testing but you can use the latest default version.

```
vault.hashicorp.com/agent-inject: 'true'
```

When set to true, this instructs the `Vault Agent Injector pod` to automatically inject Vault Agent containers into the Pod being deployed, which includes this annotation. This enables applications running in Kubernetes to access secrets stored in HashiCorp Vault.

```
vault.hashicorp.com/role: 'internal-app'
```
This configures the Vault role used by the Vault Agent auto-auth method. This is the auth role we created earlier inside `vault-0` pod. Required when vault.hashicorp.com/agent-configmap is not set.
```
vault.hashicorp.com/agent-inject-secret-tls.crt: 'pki/issue/2024-servers'
```
This configures Vault Agent to retrieve the secrets from Vault server that is required by the container. The name of the secret is any unique string after vault.hashicorp.com/agent-inject-secret-, such as 
vault.hashicorp.com/agent-inject-secret-tls.crt. The value is the path in Vault server where the secret is located. On this scenario, I'm retireving the tls.crt file created by the `agent-inject-template-` annotation.

```
vault.hashicorp.com/agent-inject-secret-tls.key: 'pki/issue/2024-servers'
```

This configures Vault Agent to retrieve the secrets from Vault server that is required by the container. The name of the secret is any unique string after vault.hashicorp.com/agent-inject-secret-, such as 
vault.hashicorp.com/agent-inject-secret-tls.key. The value is the path in Vault server where the secret is located. On this scenario, I'm retireving the tls.key file created by the `agent-inject-template-` annotation.

```
vault.hashicorp.com/agent-inject-template-tls.crt: |
  {{- with secret "pki/issue/2024-servers" "common_name=myserver.example.com" "ttl=72h" -}}
  {{ .Data.certificate }}
  {{ range .Data.ca_chain}}
  {{ . }}
  {{ end }}
  {{- end }}
```
This configures the template Vault Agent should use for rendering/writing a secret. The name of the template is any unique string after vault.hashicorp.com/agent-inject-template-, such as vault.hashicorp.com/agent-inject-template-tls.crt. This should map to the same unique value provided in vault.hashicorp.com/agent-inject-secret-. For this case, I'm creating a secret/certificate on the path `pki/issue/2024-servers` with a common name of `myserver.example.com`. The template then extracts, the certificate base64 data. See reference: [https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent/template](https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent/template). 

```
vault.hashicorp.com/agent-inject-template-tls.key: |
  {{- with secret "pki/issue/2024-servers" "common_name=myserver.example.com" "ttl=72h" -}}
  {{ .Data.private_key }}
  {{- end }}
```

Same with above, this is used to rendering/writing a secret. The name of the template is any unique string after vault.hashicorp.com/agent-inject-template-, such as vault.hashicorp.com/agent-inject-template-tls.key. This should map to the same unique value provided in vault.hashicorp.com/agent-inject-secret-. For this case, I'm creating a secret/certificate on the path `pki/issue/2024-servers` with a common name of `myserver.example.com`. The template then extracts, the private key base64 data. See reference: [https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent/template](https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent/template). 
> [!CAUTION]
> The goal of this is to get the private key data and store it into a secret. 
```
vault.hashicorp.com/secret-volume-path: /etc/secrets
```
This configures where on the filesystem (filesystem of the containers within the pod, which include NGINX web app and Vault Agent in this case) a secret will be rendered. To map a path to a specific secret, use the same unique secret name: vault.hashicorp.com/secret-volume-path-SECRET-NAME. For example, if a secret annotation vault.hashicorp.com/agent-inject-secret-foobar is configured, vault.hashicorp.com/secret-volume-path-foobar would specify where that secret is rendered. If no secret name is provided, this sets the default for all rendered secrets in the pod. In this scenario, I've set all rendered secrets to be stored in the location /etc/secrets, so I should see both tls.crt and tls.key there.

---
**References:**
- https://developer.hashicorp.com/vault/docs/platform/k8s/injector/annotations
- https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-sidecar
- https://www.hashicorp.com/blog/certificate-management-with-vault
- https://developer.hashicorp.com/vault/tutorials/secrets-management/pki-engine
- https://8gwifi.org/docs/kube-nginx.jsp
- https://medium.com/@seifeddinerajhi/securely-inject-secrets-to-pods-with-the-vault-agent-injector-3238eb774342
- https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent/template
