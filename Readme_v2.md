Vault + Kubernetes Agent Injector (Sidecar) Integration

This project demonstrates how to use HashiCorp Vault and its Agent Injector to securely inject secrets into a Kubernetes pod using the sidecar pattern.

🔐 What is Vault Agent Injector?

The Vault Agent Injector is a Kubernetes mutating admission webhook that automatically injects a Vault Agent sidecar into your pod. This agent:

Authenticates to Vault using the pod's service account

Fetches secrets defined via annotations

Writes them to a shared volume accessible by your application container

🏠 Architecture Summary

                        +---------------------------+
                        |       Vault Server        |
                        |      (vault-0 pod)        |
                        +---------------------------+
                                   ^
                                   |
                           (Authenticated via JWT)
                                   |
+----------------------------+     v     +----------------------------+
|  Application Pod           |<--- Sidecar --->|  Vault Agent Injector Pod |
|                            |                +----------------------------+
|  Container: busybox        |
|  Mount: /vault/secrets     |  <-- config.txt from Vault
+----------------------------+

🚀 How It Works

Vault is installed using the Helm chart (with injector enabled).

Kubernetes pods annotated with Vault-specific annotations get a sidecar Vault Agent.

Vault authenticates the pod using its service account JWT.

If role and policies match, secrets are injected to /vault/secrets/config.txt.

📊 Key Concepts

Vault Pod: Runs the Vault server.

Vault Role (myapp): Binds Kubernetes service accounts to Vault policies.

Service Account (vault-auth): Used by pods to authenticate with Vault.

Secret Path: Where the secret is stored in Vault, e.g. secret/data/myapp/config.

Agent Sidecar: A container injected into your app pod to fetch secrets.

📢 Important Annotations

annotations:
  vault.hashicorp.com/role: "myapp"
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/agent-inject-secret-config.txt: "secret/data/myapp/config"

Annotation

Purpose

vault.hashicorp.com/role

Role in Vault to use for authentication

vault.hashicorp.com/agent-inject

Enables injector sidecar

vault.hashicorp.com/agent-inject-secret-<filename>

Tells Vault which secret to inject

💰 Secret Location

In Vault: secret/data/myapp/config

Inside the pod: /vault/secrets/config.txt

Example content of config.txt:

username = admin
password = secret123

⚙️ How Vault Knows What to Trust

Vault uses the Kubernetes JWT token and the service account name to:

Authenticate the pod

Check if it's allowed using the Vault role (myapp)

Enforce policies (like myapp-kv-ro) to determine secret access

✍️ How to Create Required Resources

1. CA Certificate

Vault reads the CA certificate from your Kubernetes cluster automatically via:

cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt > ca.crt

2. Kubernetes Host & JWT Token

export VAULT_ADDR='http://127.0.0.1:8200'
export SA_NAME=vault-auth
export SA_NAMESPACE=default
vault write auth/kubernetes/config \
  token_reviewer_jwt=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token) \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

🚫 Common Errors

Error

Fix

403 permission denied

Ensure the Vault role is correctly bound to the SA and namespace

container not found (\"demo\")

Ensure container name matches and is up and running

no such file or directory: /vault/secrets/config.txt

Ensure secret is correctly injected and the policy grants access

🪩 What is BusyBox?

BusyBox is a lightweight Linux distribution often used in containers for debugging or testing purposes. It includes common Unix utilities like sh, ls, cat, etc.