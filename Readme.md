Vault Agent Injector with Kubernetes - Cheat Sheet & README

🚀 Overview

This project demonstrates how to integrate HashiCorp Vault with Kubernetes using the Vault Agent Injector (sidecar approach) to auto-inject secrets into pods.

🧰 Prerequisites

Kubernetes cluster (e.g., Minikube)

kubectl configured

Vault installed using Helm

helm installed

vault CLI installed

📦 Setup Steps

1. Add and Install Vault via Helm

helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm install vault hashicorp/vault \
  --namespace default \
  --set "injector.enabled=true" \
  --set "server.dev.enabled=true"

2. Port Forward Vault UI (optional)

kubectl port-forward vault-0 8200:8200

Visit: http://127.0.0.1:8200

3. Enable Kubernetes Auth in Vault

kubectl exec -it vault-0 -- sh
export VAULT_ADDR='http://127.0.0.1:8200'
vault login root
vault auth enable kubernetes

vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

4. Create a Vault Policy

# myapp-kv-ro.hcl
path "secret/data/myapp/*" {
  capabilities = ["read"]
}

vault policy write myapp-kv-ro myapp-kv-ro.hcl

5. Create a Role for the App

vault write auth/kubernetes/role/myapp \
  bound_service_account_names=vault-auth \
  bound_service_account_namespaces=default \
  policies=myapp-kv-ro \
  ttl=24h

6. Create the Secret

vault kv put secret/myapp/config username=admin password=secret123

7. Create Kubernetes Service Account

kubectl create sa vault-auth

8. Deploy Pod with Annotations

vault-sidecar-demo.yaml

apiVersion: v1
kind: Pod
metadata:
  name: vault-demo
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "myapp"
    vault.hashicorp.com/agent-inject-secret-config.txt: "secret/data/myapp/config"
spec:
  serviceAccountName: vault-auth
  containers:
    - name: demo
      image: busybox
      command: ["sleep", "3600"]

kubectl apply -f vault-sidecar-demo.yaml

9. Verify Injected Secret

kubectl exec -it vault-demo -c demo -- cat /vault/secrets/config.txt

✅ You should see:

username = admin
password = secret123

🧪 Debug Tips

If 403 permission denied, verify:

Role is bound to correct ServiceAccount

Policy allows access to secret path

If cat: No such file or directory, make sure you're using PowerShell or escape paths in Git Bash.

📁 Files

vault-sidecar-demo.yaml — Pod manifest with annotations

myapp-kv-ro.hcl — Vault policy file

🧠 Concepts Covered

Vault Kubernetes authentication

Vault Agent Injector sidecar

Vault policy management

Secret injection into pods

📎 Author

Created by Sathya with guidance from Gemini (ChatGPT)

🔗 References

https://developer.hashicorp.com/vault/docs

https://developer.hashicorp.com/vault/docs/platform/k8s/injector

