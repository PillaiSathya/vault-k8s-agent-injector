 Vault Agent Injector with Kubernetes - Cheat Sheet & Flow
🌟 What Is Vault Agent Injector?
Question	Answer
1. What is Vault Agent Injector?	A sidecar container that runs next to your app container, authenticates to Vault, fetches secrets, and writes them as files inside the pod.
2. How does the pod get the Vault token?	It uses the Kubernetes Service Account token (like vault-auth) to log in to Vault via the Kubernetes auth method.
3. Where is the secret stored inside the container?	As a file at /vault/secrets/config.txt (based on your annotation key).
4. How does Vault know which app to trust?	Through auth/kubernetes/role, which binds a Kubernetes ServiceAccount and Namespace to a Vault policy.

🛠️ Configuration Essentials
✅ 1. Get Kubernetes Host, CA Cert, JWT Token
bash
Copy
Edit
# Create a Service Account
kubectl create sa vault-auth

# Get Kubernetes host
K8S_HOST="https://kubernetes.default.svc"

# Get CA Certificate and save to file
kubectl get configmap -n kube-public kube-root-ca.crt -o jsonpath='{.data.ca\.crt}' > ca.crt

# Get Service Account JWT Token and save to file
kubectl create token vault-auth --duration=24h > reviewer-token.txt
✅ 2. Configure Kubernetes Auth in Vault
bash
Copy
Edit
vault auth enable kubernetes

vault write auth/kubernetes/config \
  token_reviewer_jwt=@reviewer-token.txt \
  kubernetes_host="$K8S_HOST" \
  kubernetes_ca_cert=@ca.crt
✅ 3. Create Vault Policy
hcl
Copy
Edit
# Save this as myapp-kv-ro.hcl
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
bash
Copy
Edit
# Apply policy
vault policy write myapp-kv-ro myapp-kv-ro.hcl
✅ 4. Create Kubernetes Auth Role
bash
Copy
Edit
vault write auth/kubernetes/role/myapp \
  bound_service_account_names=vault-auth \
  bound_service_account_namespaces=default \
  policies=myapp-kv-ro \
  ttl=24h
✅ 5. Store Secret in Vault
bash
Copy
Edit
vault kv put secret/myapp/config username=admin password=secret123
✅ 6. Annotated Pod YAML Example
yaml
Copy
Edit
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
✅ Verification
Check inside the pod:

bash
Copy
Edit
kubectl exec -it vault-demo -c demo -- cat /vault/secrets/config.txt
You should see:

text
Copy
Edit
username = admin
password = secret123
Would you like this auto-committed to your GitHub repo with a commit message like Added complete README for Vault Agent Injector demo?






You said:
sure
ChatGPT said:








Certainly, Sathya! Here's a comprehensive README.md tailored for your GitHub repository, incorporating the Vault Agent Injector summary along with detailed steps on creating the CA certificate, JWT token, and configuring the Kubernetes host.

🔐 Vault Agent Injector with Kubernetes - Complete Guide
📘 Overview
This guide provides a step-by-step walkthrough for integrating HashiCorp Vault with Kubernetes using the Vault Agent Injector. It covers the setup of Kubernetes authentication, policy creation, secret injection, and pod configuration to securely manage secrets within your Kubernetes cluster.

🌟 What is Vault Agent Injector?
Question	Answer
1. What is Vault Agent Injector?	A sidecar container that runs alongside your application pod. It authenticates to Vault, fetches secrets, and writes them to the pod as files.
2. How does the pod get the Vault token?	The agent uses the Kubernetes Service Account token (vault-auth) mounted in the pod to authenticate with Vault’s Kubernetes auth method and obtains a Vault token.
3. Where is the secret stored inside the container?	In the path /vault/secrets/ as a file. For example, secret/data/myapp/config becomes /vault/secrets/config.txt.
4. How does Vault know which app to trust?	By matching the pod’s Service Account and Namespace with a Vault Kubernetes Auth Role (bound_service_account_names and bound_service_account_namespaces).

🛠️ Configuration Steps
1. Create Kubernetes Service Account
bash
Copy
Edit
kubectl create serviceaccount vault-auth
2. Retrieve Kubernetes Host, CA Certificate, and JWT Token
bash
Copy
Edit
# Set Kubernetes host
K8S_HOST="https://kubernetes.default.svc"

# Retrieve and save the CA certificate
kubectl get configmap -n kube-public kube-root-ca.crt -o jsonpath='{.data.ca\.crt}' > ca.crt

# Retrieve and save the Service Account JWT token
kubectl create token vault-auth --duration=24h > reviewer-token.txt
3. Configure Kubernetes Authentication in Vault
bash
Copy
Edit
vault auth enable kubernetes

vault write auth/kubernetes/config \
  token_reviewer_jwt=@reviewer-token.txt \
  kubernetes_host="$K8S_HOST" \
  kubernetes_ca_cert=@ca.crt
4. Create Vault Policy
Create a policy file named myapp-kv-ro.hcl with the following content:

hcl
Copy
Edit
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
Apply the policy:

bash
Copy
Edit
vault policy write myapp-kv-ro myapp-kv-ro.hcl
5. Create Kubernetes Auth Role in Vault
bash
Copy
Edit
vault write auth/kubernetes/role/myapp \
  bound_service_account_names=vault-auth \
  bound_service_account_namespaces=default \
  policies=myapp-kv-ro \
  ttl=24h
6. Store Secret in Vault
bash
Copy
Edit
vault kv put secret/myapp/config username=admin password=secret123
7. Deploy Annotated Pod
Create a YAML file named vault-demo.yaml with the following content:

yaml
Copy
Edit
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
Apply the configuration:

bash
Copy
Edit
kubectl apply -f vault-demo.yaml
8. Verify Secret Injection
bash
Copy
Edit
kubectl exec -it vault-demo -c demo -- cat /vault/secrets/config.txt
Expected output:

ini
Copy
Edit
username = admin
password = secret123
📌 Notes
Ensure that the Vault Agent Injector is properly installed and configured in your Kubernetes cluster.

The annotations in the pod definition are crucial for the injector to function correctly.

Replace default with your desired namespace if you're operating outside the default namespace.