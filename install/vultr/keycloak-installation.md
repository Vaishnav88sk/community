# Deploying Keycloak on Vultr Kubernetes Engine (VKE)

This guide provides instructions on deploying **Keycloak** as the Identity and Access Management (IAM) solution for Microcks on **Vultr Kubernetes Engine (VKE)**, utilizing **Vultr Managed Databases for PostgreSQL** as the backend database.

## Prerequisites

1. A running **Vultr Kubernetes Engine (VKE)** cluster.
2. A **Vultr Managed Database for PostgreSQL** cluster.
3. `kubectl` installed and configured to connect to your VKE cluster.
4. `helm` installed on your local machine.

---

## 1. Prepare the PostgreSQL Database

Before deploying Keycloak, you need to create a dedicated database and user within your Vultr Managed PostgreSQL instance.

### 1.1 Connect to Vultr Managed PostgreSQL

Retrieve your database connection details (Host, Port, Username, and Password) from the **Vultr Control Panel** under your Managed Database settings.

Connect to the database using `psql`:

```sh
psql -h <VULTR_DB_HOST>.vultrdb.com -p <VULTR_DB_PORT> -U <VULTR_DB_USERNAME> defaultdb
```

### 1.2 Create Database and User

Execute the following SQL commands to create the `keycloak_db` database and a dedicated user for Keycloak to connect with:

```sql
CREATE DATABASE keycloak_db;
CREATE USER microcks WITH PASSWORD 'microcks123';
GRANT ALL PRIVILEGES ON DATABASE keycloak_db TO microcks;
ALTER DATABASE keycloak_db OWNER TO microcks;
\q
```

**IMPORTANT:** Ensure you use a strong, secure password in a production environment instead of `microcks123`.

---

## 2. Deploy Keycloak using Helm

We will use the official Bitnami Keycloak Helm chart to deploy Keycloak onto your VKE cluster.

### 2.1 Add the Bitnami Helm Repository

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### 2.2 Create the `keycloak.yaml` Configuration

Create a file named `keycloak.yaml` with the following configuration. Replace `<YOUR-DOMAIN>` with your actual domain and update the `externalDatabase` section with your Vultr database details.

```yaml
auth:
  adminUser: admin
  adminPassword: <KEYCLOAK-ADMIN-PASSWORD>

# Configure Keycloak to use the Vultr Managed PostgreSQL database
postgresql:
  enabled: false
  
externalDatabase:
  host: "<VULTR_DB_HOST>.vultrdb.com"
  port: <VULTR_DB_PORT>
  database: "keycloak_db"
  user: "microcks"
  password: "microcks123"

# Setup Ingress for Keycloak access
ingress:
  enabled: true
  ingressClassName: nginx
  hostname: keycloak.<YOUR-DOMAIN>.com
  annotations:
    nginx.ingress.kubernetes.io/proxy-buffer-size: "128k"
  tls: false
```

### 2.3 Install Keycloak

Deploy the Keycloak chart using the values file you just created:

```sh
helm install keycloak bitnami/keycloak \
  --namespace keycloak \
  --create-namespace \
  -f keycloak.yaml
```

### 2.4 Verify the Deployment

Check the status of your Keycloak pods to ensure they are running smoothly:

```sh
kubectl get pods -n keycloak -w
```

Once the pods reach the `Running` state, your Keycloak instance is successfully deployed and connected to your Vultr Managed PostgreSQL database. You can now proceed to deploy Microcks.
