

# 🔐 Secure Secrets Management with HashiCorp Vault

## 📋 Project Description

Manage application secrets **securely** using **HashiCorp Vault** integrated with **Kubernetes**.  
This project demonstrates how to:

- Install Vault on Kubernetes
- Authenticate Kubernetes workloads with Vault
- Inject dynamic secrets into a Node.js application
- Use policies to control access to secrets
- Follow security best practices for secret management

---

## 🏗️ Project Structure

```bash
secure-secrets-vault/
├── manifests/
│   ├── vault-deployment.yaml
│   ├── vault-service.yaml
│   ├── vault-configmap.yaml
│   ├── app-deployment.yaml
│   ├── app-service.yaml
│   ├── serviceaccount.yaml
├── policies/
│   ├── app-policy.hcl
├── app/
│   ├── app.js
│   ├── package.json
│   └── vault-agent-config.hcl
├── setup.sh
└── README.md
```

---

## file-setup.py

```python
import os

# Base directory
base_dir = "/home/lilia/VIDEOS/secure-secrets-vault"

# File definitions
files = {
    "manifests/vault-deployment.yaml": """apiVersion: apps/v1
kind: Deployment
metadata:
  name: vault
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vault
  template:
    metadata:
      labels:
        app: vault
    spec:
      containers:
      - name: vault
        image: vault:1.14.0
        ports:
        - containerPort: 8200
        env:
        - name: VAULT_DEV_ROOT_TOKEN_ID
          value: \"root\"
        - name: VAULT_DEV_LISTEN_ADDRESS
          value: \"0.0.0.0:8200\"
""",

    "manifests/vault-service.yaml": """apiVersion: v1
kind: Service
metadata:
  name: vault
spec:
  ports:
  - port: 8200
    targetPort: 8200
  selector:
    app: vault
  type: ClusterIP
""",

    "manifests/vault-configmap.yaml": """# optional configmap placeholder
""",

    "manifests/app-deployment.yaml": """apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      serviceAccountName: app-sa
      containers:
      - name: app
        image: node:18
        command: [\"node\", \"app.js\"]
        volumeMounts:
        - name: vault-secrets
          mountPath: /vault/secrets
          readOnly: true
      volumes:
      - name: vault-secrets
        emptyDir: {}
""",

    "manifests/app-service.yaml": """apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  selector:
    app: app
  ports:
  - protocol: TCP
    port: 3000
    targetPort: 3000
  type: ClusterIP
""",

    "manifests/serviceaccount.yaml": """apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
rules:
- apiGroups: [\"\"]
  resources: [\"secrets\"]
  verbs: [\"get\"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
subjects:
- kind: ServiceAccount
  name: app-sa
roleRef:
  kind: Role
  name: app-role
  apiGroup: rbac.authorization.k8s.io
""",

    "policies/app-policy.hcl": """path \"secret/data/app/*\" {
  capabilities = [\"read\"]
}
""",

    "app/app.js": """const express = require('express');
const fs = require('fs');
const app = express();

const secretFile = '/vault/secrets/db-creds.json';

let dbCredentials = {
  username: '',
  password: ''
};

try {
  const data = fs.readFileSync(secretFile, 'utf8');
  const secrets = JSON.parse(data);
  dbCredentials.username = secrets.data.username;
  dbCredentials.password = secrets.data.password;
} catch (error) {
  console.error('Error reading secrets:', error);
}

app.get('/', (req, res) => {
  res.send(`Connected with dynamic user: ${dbCredentials.username}`);
});

app.listen(3000, () => {
  console.log('App running on port 3000');
});
""",

    "app/package.json": """{
  \"name\": \"vault-secrets-app\",
  \"version\": \"1.0.0\",
  \"main\": \"app.js\",
  \"dependencies\": {
    \"express\": \"^4.18.2\"
  }
}
""",

    "app/vault-agent-config.hcl": """exit_after_auth = false

auto_auth {
  method \"kubernetes\" {
    mount_path = \"auth/kubernetes\"
    config = {
      role = \"app-role\"
    }
  }
  sink \"file\" {
    config = {
      path = \"/vault/secrets/db-creds.json\"
    }
  }
}
""",

    "setup.sh": """#!/bin/bash

# Apply Vault Deployment and Service
kubectl apply -f manifests/vault-deployment.yaml
kubectl apply -f manifests/vault-service.yaml

# Port-forward Vault
kubectl port-forward svc/vault 8200:8200 &

# Wait for Vault to be accessible
sleep 5

# Enable Kubernetes auth
vault auth enable kubernetes
vault write auth/kubernetes/config \\
    token_reviewer_jwt=\"$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)\" \\
    kubernetes_host=\"https://kubernetes.default.svc\"

# Create Vault Policy
vault policy write app-policy policies/app-policy.hcl

# Store Dynamic Secrets
vault kv put secret/app/db username=\"dbuser\" password=\"dbpass\"

# Deploy Application
kubectl apply -f manifests/serviceaccount.yaml
kubectl apply -f manifests/app-deployment.yaml
kubectl apply -f manifests/app-service.yaml

# Port-forward App
kubectl port-forward svc/app 3000:3000 &

echo \"Setup Complete. App available at http://localhost:3000\"
""",

    "README.md": """# 🔐 Secure Secrets Management with HashiCorp Vault

## 📋 Project Description

Manage application secrets securely using HashiCorp Vault integrated with Kubernetes.

...

# (Put the complete markdown README here if you want)
"""
}

# Create directories and files
for relative_path, content in files.items():
    file_path = os.path.join(base_dir, relative_path)
    os.makedirs(os.path.dirname(file_path), exist_ok=True)
    with open(file_path, 'w') as f:
        f.write(content)

print(f"✅ All files created successfully inside {base_dir}")


```

---

## 🎯 Project Objectives

- 🚀 Install and configure **Vault** in Kubernetes
- 🔑 Enable **Kubernetes authentication** in Vault
- 🧩 Create and enforce **Vault policies** for fine-grained access control
- 💾 Inject **dynamic database credentials** into a running **Node.js application**
- 🛡️ Avoid hardcoding sensitive credentials inside the app

---

## ⚙️ Prerequisites

- Kubernetes cluster (Minikube, Kind, EKS, AKS, GKE, etc.)
- `kubectl` installed and configured
- Vault CLI installed (`brew install vault` or download from [HashiCorp](https://developer.hashicorp.com/vault/downloads))
- Node.js installed for local development (optional)

---

## 📜 Step-by-Step Setup

---

### 🥇 1. Install Vault on Kubernetes

Deploy Vault server inside Kubernetes:

```bash
kubectl apply -f manifests/vault-deployment.yaml
kubectl apply -f manifests/vault-service.yaml
```

Port-forward Vault service:

```bash
kubectl port-forward svc/vault 8200:8200
```

Access Vault UI:  
🔗 [http://localhost:8200](http://localhost:8200)  
Login Token: `root`

---

### 🥈 2. Enable Kubernetes Authentication in Vault

Enable Kubernetes auth method:

```bash
vault auth enable kubernetes
```

Configure Vault to talk to Kubernetes API:

```bash
vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://kubernetes.default.svc"
```

---

### 🥉 3. Create Vault Policy for the App

Define the app policy:

```hcl
# policies/app-policy.hcl
path "secret/data/app/*" {
  capabilities = ["read"]
}
```

Apply the policy in Vault:

```bash
vault policy write app-policy policies/app-policy.hcl
```

---

### 🏅 4. Create Kubernetes Service Account

Create a service account and RBAC permissions:

```bash
kubectl apply -f manifests/serviceaccount.yaml
```

---

### 🏆 5. Deploy the Node.js Application

Configure the app deployment to use Vault Agent Injector:

```bash
kubectl apply -f manifests/app-deployment.yaml
kubectl apply -f manifests/app-service.yaml
```

---

### 🎯 6. Store Secrets in Vault

Inject dynamic database credentials:

```bash
vault kv put secret/app/db username="dbuser" password="dbpass"
```

---

### 🛠️ 7. Access the Application

Port-forward the app service:

```bash
kubectl port-forward svc/app 3000:3000
```

Access it in the browser:  
🔗 [http://localhost:3000](http://localhost:3000)

✅ It should display the **Vault-injected dynamic credentials**!

---

## 🧹 CLI Quick Summary

```bash
# Deploy Vault
kubectl apply -f manifests/vault-deployment.yaml
kubectl apply -f manifests/vault-service.yaml

# Port-forward Vault
kubectl port-forward svc/vault 8200:8200

# Configure Vault Auth
vault auth enable kubernetes
vault write auth/kubernetes/config token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" kubernetes_host="https://kubernetes.default.svc"

# Apply Vault Policy
vault policy write app-policy policies/app-policy.hcl

# Save Secrets
vault kv put secret/app/db username="dbuser" password="dbpass"

# Deploy App
kubectl apply -f manifests/serviceaccount.yaml
kubectl apply -f manifests/app-deployment.yaml
kubectl apply -f manifests/app-service.yaml

# Port-forward App
kubectl port-forward svc/app 3000:3000
```

---

## 📈 Key Skills Practiced

- ✅ HashiCorp Vault installation and configuration
- ✅ Kubernetes authentication with Vault
- ✅ Vault dynamic secrets injection
- ✅ Vault policies and access control
- ✅ Secure ServiceAccount-based access
- ✅ Node.js app secret management best practices

---

## 📚 References

- [Vault Kubernetes Auth Method](https://developer.hashicorp.com/vault/docs/auth/kubernetes)
- [Vault Dynamic Secrets](https://developer.hashicorp.com/vault/docs/secrets/databases)
- [Kubernetes ServiceAccount](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)

---

# 🚀 Happy Secure Deployment!

---

Would you also like me to create a **setup.sh** script that automates all the CLI steps for you (bonus for a demo)? 🎯  
Would you like that too? 🚀  
It would make running the project even faster! 🔥  
(Just one script to setup everything)
