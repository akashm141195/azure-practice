# Azure Three-Tier Application Workshop
### End-to-End Hands-On Tutorial

---

## Overview

In this workshop, you will deploy a three-tier application on Azure covering:
- Virtual Networks and Subnets
- Network Security Groups (NSGs) — subnet level and NIC level
- Virtual Machines per tier
- Load Balancer for frontend (no public IP on VM)
- Managed Identities for secure authentication
- Azure Key Vault for secrets
- Azure Storage Account for file access

**Duration:** 3 hours  
**Audience:** Beginners to intermediate Azure users  
**Setup:** One Azure subscription, one resource group per attendee

---

## Architecture

```
Internet
   │
   ▼
[Load Balancer]  ← only public IP in the setup
   │  (L4 — transparent, VM sees real user IP)
   ▼
[Frontend Subnet] ── Subnet NSG + NIC NSG (allow HTTP from Internet/myIP)
   │  VM: Nginx (no public IP — SSH via copied private key)
   ▼
[App Subnet] ── NSG (allow inbound only from Frontend Subnet)
   │  VM: FastAPI
   ▼
[Database Subnet] ── NSG (allow inbound only from App Subnet)
      VM: PostgreSQL

Key Vault   ◄─── App VM (via Managed Identity, outbound)
Storage     ◄─── App VM (via Managed Identity, outbound)
```

> **Load Balancer vs App Gateway — NSG source difference:**
> - Azure Load Balancer (L4): transparent — VM sees the **real user IP**. NSG source = `Internet` or user IP.
> - Azure App Gateway (L7): terminates connection — VM sees the **App Gateway's private IP**. NSG source = App Gateway subnet CIDR.

---

## Pre-requisites

- Azure free tier account (one subscription for instructor)
- Nine resource groups pre-created (one per attendee)
- Each attendee assigned **Contributor** role scoped to their resource group
- Azure CLI or Azure Portal access
- Git Bash (Windows) or Terminal (Mac/Linux)

> **Git Bash on Windows:** Prefix `az role assignment create` commands with `MSYS_NO_PATHCONV=1` to prevent path mangling:
> ```bash
> MSYS_NO_PATHCONV=1 az role assignment create \
>   --assignee "<principalId>" \
>   --role "Key Vault Secrets User" \
>   --scope "/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.KeyVault/vaults/<vault>"
> ```

---

## Step 1 — Create a Resource Group

Each attendee works inside their own resource group.

```bash
az group create \
  --name rg-workshop-user01 \
  --location eastus
```

**Portal:** Search "Resource Groups" → Create → Name it → Select region → Review + Create

---

## Step 2 — Create a Virtual Network with Three Subnets

One VNet with three isolated subnets — one per tier.

```bash
az network vnet create \
  --resource-group rg-workshop-user01 \
  --name vnet-workshop \
  --address-prefix 10.0.0.0/16 \
  --subnet-name subnet-frontend \
  --subnet-prefix 10.0.1.0/24
```

Add App and DB subnets:

```bash
az network vnet subnet create \
  --resource-group rg-workshop-user01 \
  --vnet-name vnet-workshop \
  --name subnet-app \
  --address-prefix 10.0.2.0/24

az network vnet subnet create \
  --resource-group rg-workshop-user01 \
  --vnet-name vnet-workshop \
  --name subnet-db \
  --address-prefix 10.0.3.0/24
```

**Key concept:**
- Azure handles routing between subnets automatically — no route tables needed for this setup.
- VNet is private by default. Traffic only enters if NSG rules allow it.

---

## Step 3 — Create Network Security Groups (NSGs)

NSGs are the traffic control layer. They decide what traffic is allowed in and out of each subnet.

### Frontend NSG — Allow HTTP/HTTPS from Internet

```bash
az network nsg create \
  --resource-group rg-workshop-user01 \
  --name nsg-frontend

az network nsg rule create \
  --resource-group rg-workshop-user01 \
  --nsg-name nsg-frontend \
  --name allow-http-inbound \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes "Internet" \
  --destination-port-ranges 80 443
```

> **Note:** You can tighten this to `--source-address-prefixes "<yourIP>"` during the workshop so only your IP can reach the frontend. Use `Internet` for open public access.

### App Tier NSG — Allow only from Frontend Subnet

```bash
az network nsg create \
  --resource-group rg-workshop-user01 \
  --name nsg-app

az network nsg rule create \
  --resource-group rg-workshop-user01 \
  --nsg-name nsg-app \
  --name allow-from-frontend \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes 10.0.1.0/24 \
  --destination-port-ranges 8080
```

### Database NSG — Allow only from App Subnet

```bash
az network nsg create \
  --resource-group rg-workshop-user01 \
  --name nsg-db

az network nsg rule create \
  --resource-group rg-workshop-user01 \
  --nsg-name nsg-db \
  --name allow-from-app \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes 10.0.2.0/24 \
  --destination-port-ranges 5432
```

### Attach NSGs to Subnets

```bash
az network vnet subnet update \
  --resource-group rg-workshop-user01 \
  --vnet-name vnet-workshop \
  --name subnet-frontend \
  --network-security-group nsg-frontend

az network vnet subnet update \
  --resource-group rg-workshop-user01 \
  --vnet-name vnet-workshop \
  --name subnet-app \
  --network-security-group nsg-app

az network vnet subnet update \
  --resource-group rg-workshop-user01 \
  --vnet-name vnet-workshop \
  --name subnet-db \
  --network-security-group nsg-db
```

**Key concept (AWS comparison):**
| AWS | Azure |
|---|---|
| Security Groups | NSGs |
| Internet Gateway | Not needed (outbound allowed by default) |
| NAT Gateway | Not needed for basic setup |
| Route Tables | Not needed within a single VNet |

---

## Step 4 — Deploy Virtual Machines

**All three VMs have no public IP.** The frontend is exposed via the Load Balancer only. SSH to the frontend VM uses the Load Balancer's public IP with SSH port forwarded, or a temporary NSG rule on the NIC.

### Frontend VM (No Public IP — exposed via Load Balancer)

```bash
az vm create \
  --resource-group rg-workshop-user01 \
  --name vm-frontend \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --vnet-name vnet-workshop \
  --subnet subnet-frontend \
  --admin-username azureuser \
  --generate-ssh-keys \
  --assign-identity \
  --public-ip-address ""
```

### App Tier VM (Private — No Public IP)

```bash
az vm create \
  --resource-group rg-workshop-user01 \
  --name vm-app \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --vnet-name vnet-workshop \
  --subnet subnet-app \
  --admin-username azureuser \
  --generate-ssh-keys \
  --assign-identity \
  --public-ip-address ""
```

### Database VM (Private — No Public IP)

```bash
az vm create \
  --resource-group rg-workshop-user01 \
  --name vm-db \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --vnet-name vnet-workshop \
  --subnet subnet-db \
  --admin-username azureuser \
  --generate-ssh-keys \
  --assign-identity \
  --public-ip-address ""
```

**Key concept:**
`--assign-identity` enables a **system-assigned Managed Identity** on the VM automatically. No manual identity creation needed.

---

## Step 4a — NIC-Level NSG for SSH Access

When you create a VM in Azure, it automatically creates an NSG attached to the **VM's NIC** (in addition to the subnet NSG). Add an inbound SSH rule on the NIC NSG so you can SSH in for the workshop.

**Portal:** Go to VM → Networking → Add inbound port rule:
- Source: `My IP` (recommended) or `Any`
- Destination port: `22`
- Protocol: TCP
- Priority: `100`
- Name: `allow-ssh`

**CLI:**
```bash
# Find the NIC NSG name (auto-created as vm-frontend-nsg or similar)
az network nsg list --resource-group rg-workshop-user01 --query "[].name" -o tsv

az network nsg rule create \
  --resource-group rg-workshop-user01 \
  --nsg-name vm-frontend-nsg \
  --name allow-ssh \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes "<yourPublicIP>" \
  --destination-port-ranges 22
```

> **Two NSGs in play:** The subnet NSG controls traffic at the subnet level for all VMs. The NIC NSG controls traffic for that specific VM. Both are evaluated — for inbound: subnet NSG first, then NIC NSG.

---

## Step 4b — SSH into App and DB VMs via Frontend (Jump Host)

Since only the Load Balancer has a public IP, SSH to app and db VMs goes **through the frontend VM** as a jump host.

**First, copy your private key to the frontend VM:**

```bash
# From your local machine — copy key to frontend VM
# (replace with your frontend VM's accessible IP or LB SSH port)
scp -i id_rsa id_rsa azureuser@<frontend-vm-ip>:/home/azureuser/

# SSH into frontend
ssh -i id_rsa azureuser@<frontend-vm-ip>

# On frontend VM — set correct permissions
chmod 400 ~/id_rsa

# SSH from frontend into app VM (using private IP)
ssh -i ~/id_rsa azureuser@10.0.2.4

# SSH from frontend into db VM (using private IP)
ssh -i ~/id_rsa azureuser@10.0.3.4
```

> **Security note:** Copying a private key to a jump host is acceptable for a workshop. In production, use Azure Bastion or SSH agent forwarding (`ssh -A`) instead.

---

## Step 5 — Set Up Load Balancer for Frontend

The Load Balancer is the **only entry point from the internet**. It has the public IP; the frontend VM does not.

```bash
# Create public IP for Load Balancer
az network public-ip create \
  --resource-group rg-workshop-user01 \
  --name pip-lb \
  --sku Standard

# Create Load Balancer
az network lb create \
  --resource-group rg-workshop-user01 \
  --name lb-frontend \
  --sku Standard \
  --public-ip-address pip-lb \
  --frontend-ip-name lb-frontend-ip \
  --backend-pool-name lb-backend-pool

# Health probe
az network lb probe create \
  --resource-group rg-workshop-user01 \
  --lb-name lb-frontend \
  --name lb-health-probe \
  --protocol Http \
  --port 80 \
  --path /

# Load Balancing rule — HTTP
az network lb rule create \
  --resource-group rg-workshop-user01 \
  --lb-name lb-frontend \
  --name lb-http-rule \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name lb-frontend-ip \
  --backend-pool-name lb-backend-pool \
  --probe-name lb-health-probe

# Add frontend VM NIC to backend pool
NIC_ID=$(az vm show \
  --resource-group rg-workshop-user01 \
  --name vm-frontend \
  --query "networkProfile.networkInterfaces[0].id" -o tsv)

az network nic ip-config address-pool add \
  --address-pool lb-backend-pool \
  --ip-config-name ipconfig1 \
  --nic-name $(basename $NIC_ID) \
  --resource-group rg-workshop-user01 \
  --lb-name lb-frontend
```

---

## Step 6 — Create Azure Key Vault and Store Secrets

> **Key Vault names are globally unique across all of Azure.** Choose a name that is unlikely to conflict — e.g. include your initials or a random suffix: `kv-workshop-akm01`.

```bash
az keyvault create \
  --name kv-workshop-akm01 \
  --resource-group rg-workshop-user01 \
  --location eastus \
  --enable-rbac-authorization true
```

### Grant yourself permission to SET secrets (Key Vault Secrets Officer)

Before you can store secrets, your **user account** needs the `Key Vault Secrets Officer` role on the vault. This is separate from the VM's Managed Identity which only needs `Key Vault Secrets User` (read-only).

```bash
# Get your own user object ID
MY_USER_ID=$(az ad signed-in-user show --query id -o tsv)

KEYVAULT_ID=$(az keyvault show \
  --name kv-workshop-akm01 \
  --resource-group rg-workshop-user01 \
  --query id -o tsv)

# Assign Secrets Officer to yourself so you can write secrets
az role assignment create \
  --assignee $MY_USER_ID \
  --role "Key Vault Secrets Officer" \
  --scope $KEYVAULT_ID
```

**Git Bash (Windows) — use MSYS_NO_PATHCONV=1:**
```bash
MSYS_NO_PATHCONV=1 az role assignment create \
  --assignee "<your-user-object-id>" \
  --role "Key Vault Secrets Officer" \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-workshop-user01/providers/Microsoft.KeyVault/vaults/kv-workshop-akm01"
```

### Store the DB password as a secret

```bash
az keyvault secret set \
  --vault-name kv-workshop-akm01 \
  --name db-password \
  --value "StrongPass@123"
```

---

## Step 7 — Create Storage Account and Upload a File

```bash
az storage account create \
  --name storeworkshop01 \
  --resource-group rg-workshop-user01 \
  --location eastus \
  --sku Standard_LRS

az storage container create \
  --name workshop-files \
  --account-name storeworkshop01

# Create a sample config file and upload it
echo '{"env":"workshop","version":"1.0"}' > config.json

az storage blob upload \
  --account-name storeworkshop01 \
  --container-name workshop-files \
  --name config.json \
  --file ./config.json
```

---

## Step 8 — Managed Identity and Role Assignment

Each VM has a system-assigned Managed Identity. Now assign permissions so the **App VM** can read from Storage and read secrets from Key Vault.

### Get the App VM's Managed Identity Principal ID

```bash
APP_VM_IDENTITY=$(az vm show \
  --resource-group rg-workshop-user01 \
  --name vm-app \
  --query identity.principalId \
  --output tsv)

echo $APP_VM_IDENTITY  # copy this value
```

### Assign Storage Blob Data Reader Role

```bash
STORAGE_ID=$(az storage account show \
  --name storeworkshop01 \
  --resource-group rg-workshop-user01 \
  --query id --output tsv)

az role assignment create \
  --assignee $APP_VM_IDENTITY \
  --role "Storage Blob Data Reader" \
  --scope $STORAGE_ID
```

### Assign Key Vault Secrets User Role to App VM

```bash
KEYVAULT_ID=$(az keyvault show \
  --name kv-workshop-akm01 \
  --resource-group rg-workshop-user01 \
  --query id --output tsv)

az role assignment create \
  --assignee $APP_VM_IDENTITY \
  --role "Key Vault Secrets User" \
  --scope $KEYVAULT_ID
```

**Git Bash (Windows):**
```bash
MSYS_NO_PATHCONV=1 az role assignment create \
  --assignee "<app-vm-principal-id>" \
  --role "Key Vault Secrets User" \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-workshop-user01/providers/Microsoft.KeyVault/vaults/kv-workshop-akm01"
```

**Role summary:**

| Who | Role | On What | Purpose |
|---|---|---|---|
| Your user account | Key Vault Secrets Officer | Key Vault | Write/manage secrets |
| App VM Managed Identity | Key Vault Secrets User | Key Vault | Read secrets at runtime |
| App VM Managed Identity | Storage Blob Data Reader | Storage Account | Read blobs at runtime |

**Managed Identity types:**

| Type | Description |
|---|---|
| System-assigned | Tied to one VM. Created and deleted with the VM. Cannot be shared. |
| User-assigned | Created as a standalone Azure resource. Can be attached to multiple VMs. Shareable. |

**AWS comparison:**
| AWS | Azure |
|---|---|
| IAM Role | Azure Role (RBAC) |
| IAM Policy | Azure Permission |
| Instance Profile | Managed Identity |

---

## Step 9 — Access Storage from App VM Using Managed Identity

SSH into the App VM (via frontend as jump host) and run:

```bash
# Get access token using Managed Identity
TOKEN=$(curl -s "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://storage.azure.com/" \
  -H "Metadata: true" | python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'])")

# Read file from Storage
curl -s "https://storeworkshop01.blob.core.windows.net/workshop-files/config.json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "x-ms-version: 2020-04-08"
```

**Traffic flow:**
1. App VM (private subnet) sends outbound request to storage endpoint
2. Azure allows outbound traffic by default (NSG outbound rules are open)
3. Storage account validates the Managed Identity token
4. Checks role assignment — Storage Blob Data Reader confirmed
5. Returns the file to the VM

---

## Step 10 — End-to-End Test

1. Hit the Load Balancer public IP in a browser → reaches Frontend VM
2. Frontend VM calls App VM on port 8080 (allowed by NSG)
3. App VM fetches DB password from Key Vault using Managed Identity
4. App VM reads config from Storage using Managed Identity
5. App VM queries Database VM on port 5432 (allowed by NSG)
6. Response flows back through App → Frontend → Browser

---

## Application Code

This section provides the actual application code to deploy on each VM tier.

---

### Database Tier — PostgreSQL Setup (vm-db)

SSH into `vm-db` (via frontend jump host) and run:

```bash
# Install PostgreSQL
sudo apt update && sudo apt install -y postgresql postgresql-contrib

# Start and enable
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Create database and user
sudo -u postgres psql <<EOF
CREATE DATABASE workshopdb;
CREATE USER appuser WITH ENCRYPTED PASSWORD 'StrongPass@123';
GRANT ALL PRIVILEGES ON DATABASE workshopdb TO appuser;
\c workshopdb
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(150) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');
INSERT INTO users (name, email) VALUES ('Bob', 'bob@example.com');
EOF
```

Configure PostgreSQL to accept connections from the App subnet:

```bash
# Allow connections from App subnet
echo "host workshopdb appuser 10.0.2.0/24 md5" | sudo tee -a /etc/postgresql/*/main/pg_hba.conf

# Listen on all interfaces
sudo sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" /etc/postgresql/*/main/postgresql.conf

sudo systemctl restart postgresql
```

---

### Backend API Tier — FastAPI App (vm-app)

SSH into `vm-app` (via frontend jump host) and run:

```bash
sudo apt update && sudo apt install -y python3-pip python3-venv
mkdir ~/app && cd ~/app
python3 -m venv venv
source venv/bin/activate
pip install fastapi uvicorn psycopg2-binary azure-keyvault-secrets azure-identity requests
```

Create the API:

```bash
sudo vi ~/app/main.py
```

Paste the following:

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from azure.identity import ManagedIdentityCredential
from azure.keyvault.secrets import SecretClient
import psycopg2
import os

app = FastAPI(title="Workshop API", version="1.0.0")

KEY_VAULT_URL = os.getenv("KEY_VAULT_URL", "https://kv-workshop-akm01.vault.azure.net/")

def get_db_connection():
    credential = ManagedIdentityCredential()
    client = SecretClient(vault_url=KEY_VAULT_URL, credential=credential)
    db_password = client.get_secret("db-password").value

    conn = psycopg2.connect(
        host=os.getenv("DB_HOST", "10.0.3.4"),
        database="workshopdb",
        user="appuser",
        password=db_password,
        connect_timeout=5
    )
    return conn

class UserCreate(BaseModel):
    name: str
    email: str

@app.get("/")
def root():
    return {"message": "Workshop API is running", "status": "healthy"}

@app.get("/health")
def health():
    return {"status": "ok"}

@app.get("/users")
def get_users():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT id, name, email, created_at FROM users ORDER BY id;")
        rows = cursor.fetchall()
        cursor.close()
        conn.close()
        return [{"id": r[0], "name": r[1], "email": r[2], "created_at": str(r[3])} for r in rows]
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/users")
def create_user(user: UserCreate):
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("INSERT INTO users (name, email) VALUES (%s, %s) RETURNING id;", (user.name, user.email))
        user_id = cursor.fetchone()[0]
        conn.commit()
        cursor.close()
        conn.close()
        return {"id": user_id, "name": user.name, "email": user.email}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.delete("/users/{user_id}")
def delete_user(user_id: int):
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("DELETE FROM users WHERE id = %s;", (user_id,))
        conn.commit()
        cursor.close()
        conn.close()
        return {"message": f"User {user_id} deleted"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

Start the API:

```bash
cd ~/app
source venv/bin/activate
DB_HOST=10.0.3.4 KEY_VAULT_URL=https://kv-workshop-akm01.vault.azure.net/ \
  uvicorn main:app --host 0.0.0.0 --port 8080 --reload
```

Run as a background service:

```bash
sudo vi /etc/systemd/system/workshop-api.service
```

Paste:

```ini
[Unit]
Description=Workshop FastAPI App
After=network.target

[Service]
User=azureuser
WorkingDirectory=/home/azureuser/app
Environment="DB_HOST=10.0.3.4"
Environment="KEY_VAULT_URL=https://kv-workshop-akm01.vault.azure.net/"
ExecStart=/home/azureuser/app/venv/bin/uvicorn main:app --host 0.0.0.0 --port 8080
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable workshop-api
sudo systemctl start workshop-api

# Verify
curl http://localhost:8080/users
```

---

### Frontend Tier — HTML + Nginx (vm-frontend)

SSH into `vm-frontend` and install Nginx:

```bash
sudo apt update && sudo apt install -y nginx
sudo mkdir -p /var/www/workshop
```

Create the HTML file using vi:

```bash
sudo vi /var/www/workshop/index.html
```

Paste the following (update the `API` constant to your **App VM's private IP**):

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Azure Workshop App</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: 'Segoe UI', sans-serif; background: #0f172a; color: #e2e8f0; min-height: 100vh; padding: 2rem; }
    h1 { color: #38bdf8; margin-bottom: 0.25rem; font-size: 1.8rem; }
    .subtitle { color: #64748b; margin-bottom: 2rem; font-size: 0.9rem; }
    .card { background: #1e293b; border-radius: 12px; padding: 1.5rem; margin-bottom: 1.5rem; border: 1px solid #334155; }
    .card h2 { color: #94a3b8; font-size: 1rem; margin-bottom: 1rem; }
    input { background: #0f172a; border: 1px solid #334155; color: #e2e8f0; padding: 0.5rem 0.75rem; border-radius: 6px; margin-right: 0.5rem; margin-bottom: 0.5rem; width: 200px; }
    button { background: #0284c7; color: white; border: none; padding: 0.5rem 1rem; border-radius: 6px; cursor: pointer; font-size: 0.9rem; }
    button:hover { background: #0369a1; }
    button.danger { background: #dc2626; }
    button.danger:hover { background: #b91c1c; }
    table { width: 100%; border-collapse: collapse; font-size: 0.9rem; }
    th { text-align: left; padding: 0.5rem; color: #64748b; border-bottom: 1px solid #334155; }
    td { padding: 0.5rem; border-bottom: 1px solid #1e293b; }
    .badge { display: inline-block; background: #0c4a6e; color: #38bdf8; padding: 2px 8px; border-radius: 99px; font-size: 0.75rem; }
    #status { color: #4ade80; margin-top: 1rem; font-size: 0.85rem; min-height: 1.2rem; }
  </style>
</head>
<body>
  <h1>Azure Three-Tier Workshop</h1>
  <p class="subtitle">Frontend → Backend API → PostgreSQL Database</p>

  <div class="card">
    <h2>ADD USER</h2>
    <input id="name" type="text" placeholder="Name" />
    <input id="email" type="email" placeholder="Email" />
    <button onclick="createUser()">Add User</button>
    <div id="status"></div>
  </div>

  <div class="card">
    <h2>USERS FROM DATABASE</h2>
    <button onclick="loadUsers()" style="margin-bottom:1rem;">Refresh</button>
    <table>
      <thead><tr><th>ID</th><th>Name</th><th>Email</th><th>Created</th><th>Action</th></tr></thead>
      <tbody id="user-table"></tbody>
    </table>
  </div>

  <script>
    // !! Update this to your App VM's private IP !!
    const API = "http://10.0.2.4:8080";

    async function loadUsers() {
      try {
        const res = await fetch(`${API}/users`);
        const users = await res.json();
        document.getElementById("user-table").innerHTML = users.map(u => `
          <tr>
            <td><span class="badge">${u.id}</span></td>
            <td>${u.name}</td>
            <td>${u.email}</td>
            <td>${new Date(u.created_at).toLocaleDateString()}</td>
            <td><button class="danger" onclick="deleteUser(${u.id})">Delete</button></td>
          </tr>`).join("");
      } catch (err) {
        document.getElementById("status").textContent = "Error: " + err.message;
      }
    }

    async function createUser() {
      const name = document.getElementById("name").value.trim();
      const email = document.getElementById("email").value.trim();
      if (!name || !email) { document.getElementById("status").textContent = "Fill both fields."; return; }
      try {
        const res = await fetch(`${API}/users`, {
          method: "POST", headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ name, email })
        });
        const data = await res.json();
        document.getElementById("status").textContent = `Created: ${data.name} (ID: ${data.id})`;
        document.getElementById("name").value = "";
        document.getElementById("email").value = "";
        loadUsers();
      } catch (err) { document.getElementById("status").textContent = "Error: " + err.message; }
    }

    async function deleteUser(id) {
      try {
        await fetch(`${API}/users/${id}`, { method: "DELETE" });
        document.getElementById("status").textContent = `Deleted user ${id}`;
        loadUsers();
      } catch (err) { document.getElementById("status").textContent = "Error: " + err.message; }
    }

    loadUsers();
  </script>
</body>
</html>
```

Configure Nginx using vi:

```bash
sudo vi /etc/nginx/sites-available/workshop
```

Paste:

```nginx
server {
    listen 80;
    server_name _;
    root /var/www/workshop;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Enable and restart:

```bash
sudo ln -s /etc/nginx/sites-available/workshop /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl restart nginx
```

---

### End-to-End Request Flow

```
User Browser
    │  HTTP GET /
    ▼
Load Balancer (Public IP) — only public entry point
    │  forwards to port 80 on vm-frontend
    │  VM sees REAL user IP (LB is L4, transparent)
    ▼
vm-frontend (Nginx) — no public IP
    │  serves index.html
    │  JS calls http://10.0.2.4:8080/users
    ▼
vm-app (FastAPI on port 8080) — private only
    │  fetches DB password from Key Vault via Managed Identity
    │  connects to PostgreSQL on 10.0.3.4:5432
    ▼
vm-db (PostgreSQL) — private only
    │  returns rows
    ▼
API → Frontend → Browser renders table
```

---

### Quick Verification Checklist

```bash
# From vm-db: confirm PostgreSQL is running
sudo systemctl status postgresql

# From vm-app: check DB connectivity directly
psql -h 10.0.3.4 -U appuser -d workshopdb -c "SELECT * FROM users;"

# From vm-app: check API is running
curl http://localhost:8080/users

# From vm-frontend: check API is reachable across subnets
curl http://10.0.2.4:8080/health

# From your machine: check frontend loads via Load Balancer
curl http://<load-balancer-public-ip>/
```

---

## Cleanup

Delete the resource group to remove everything and avoid charges:

```bash
az group delete \
  --name rg-workshop-user01 \
  --yes \
  --no-wait
```

Deleting the resource group removes all VMs, VNets, NSGs, Storage, Key Vault, Load Balancer, and Managed Identities inside it.

**Instructor cleanup — delete all nine resource groups at once:**

```bash
for i in $(seq -w 1 9); do
  az group delete --name rg-workshop-user0$i --yes --no-wait
done
```

---

## Key Concepts Summary

| Concept | What it does |
|---|---|
| VNet | Private network in Azure |
| Subnet | Segment of the VNet per tier |
| Subnet NSG | Traffic control for all VMs in a subnet |
| NIC NSG | Traffic control for one specific VM |
| Load Balancer (L4) | Distributes internet traffic; VM sees real user IP |
| App Gateway (L7) | Terminates connection; VM sees App GW private IP |
| System-assigned Managed Identity | Auto-created identity tied to one VM |
| User-assigned Managed Identity | Standalone identity, shareable across VMs |
| Key Vault Secrets Officer | Role needed to write/manage secrets |
| Key Vault Secrets User | Role needed to read secrets (given to VM identity) |
| Storage Blob Data Reader | Role needed to read blobs (given to VM identity) |
| Key Vault name | Globally unique across all of Azure |
| Resource Group | Container for all resources — delete it, delete everything |

---

*Workshop prepared for Azure Fundamentals — 3-hour hands-on lab*
