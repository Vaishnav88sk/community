# Deploying Microcks on Vultr Cloud

## Overview
This guide provides a step-by-step approach to deploying **Microcks** on **Vultr Cloud**. We will utilize **Vultr Kubernetes Engine (VKE)** for container orchestration, **Vultr Managed Databases for PostgreSQL** for Keycloak, and a highly available in-cluster **MongoDB** deployment for Microcks data storage.

## Prerequisites

Ensure the following tools are installed on your local system:

1. **Vultr CLI (`vultr-cli`):** [Install Guide](https://github.com/vultr/vultr-cli)
2. **kubectl (Kubernetes CLI):** [Install Guide](https://kubernetes.io/docs/tasks/tools/)
3. **Helm:** [Install Guide](https://helm.sh/docs/intro/install/)
4. **An active Vultr account** with billing enabled.

---

## 1. Configure Vultr CLI

Authenticate your CLI with your Vultr API Key (which can be generated in the Vultr Control Panel under Account -> API):

```sh
export VULTR_API_KEY="your-vultr-api-key"
```

## 2. Create a Vultr Kubernetes Engine (VKE) Cluster

Provision a VKE cluster using the Vultr CLI. We will deploy a cluster with 3 nodes using standard compute instances.

```sh
vultr-cli kubernetes create \
  --label "microcks-cluster" \
  --region "ewr" \
  --version "v1.30.0+1" \
  --node-pools "quantity:3,plan:vc2-2c-4gb,label:microcks-pool"
```

> Note: You can find available regions using `vultr-cli regions list` and available plans using `vultr-cli plans list`.

Wait for the cluster to finish provisioning (this usually takes 5-10 minutes). Once it is active, download the `kubeconfig` to interact with your cluster:

```sh
# Retrieve the Cluster ID
vultr-cli kubernetes list

# Download the Kubeconfig
vultr-cli kubernetes config <CLUSTER-ID> > ~/.kube/config
```

Verify your cluster connectivity:
```sh
kubectl get nodes
```

## 3. Provision Vultr Managed PostgreSQL

Microcks relies on Keycloak for identity management, which requires a PostgreSQL database.

Create a Managed PostgreSQL cluster:

```sh
vultr-cli database create \
  --database-engine pg \
  --label "microcks-postgres" \
  --region "ewr" \
  --plan "vdc-1c-4gb"
```

Retrieve your database connection details (Host, Port, Username, and Password) from the Vultr Control Panel. You will need these to configure Keycloak.

## 4. Deploy Keycloak on VKE

With your Kubernetes cluster and PostgreSQL database ready, deploy Keycloak. 

Follow the **[Keycloak Deployment Guide for Vultr](keycloak-installation.md)** to complete this step.

Once Keycloak is successfully deployed, ensure you:
- Create a `microcks` **realm**.
- Set up a `microcks` **client**.
- Create a `microcks` **user** with appropriate access roles.

## 5. Deploy MongoDB on VKE

Because Vultr does not offer a native managed MongoDB service, we will deploy a highly-available MongoDB instance directly into your VKE cluster utilizing Vultr's native Block Storage for persistence.

### 5.1 Add the Bitnami Repository
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### 5.2 Install MongoDB
Deploy MongoDB into the `microcks` namespace:

```sh
helm install mongodb bitnami/mongodb -n microcks --create-namespace \
  --set architecture=standalone \
  --set auth.enabled=true \
  --set auth.rootPassword=strongRootPassword123 \
  --set auth.username=microcks \
  --set auth.password=microcks123 \
  --set auth.database=microcks \
  --set persistence.enabled=true \
  --set persistence.size=20Gi
```

### 5.3 Create MongoDB Connection Secret
Securely store your MongoDB credentials in a Kubernetes Secret so the Microcks application can mount them.

```sh
kubectl create secret generic microcks-mongodb-connection -n microcks \
  --from-literal=username=microcks \
  --from-literal=password=microcks123
```

## 6. Deploy Microcks using Helm

### 6.1 Install NGINX Ingress Controller
Vultr automatically provisions an external Load Balancer when an Ingress Controller is deployed.

```sh
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

### 6.2 Prepare `microcks.yaml` Configuration File
Create the `microcks.yaml` file with the configuration below. Ensure you replace `<YOUR-DOMAIN>` with your actual domain name.

```yaml
appName: microcks

microcks:
  url: microcks.<YOUR-DOMAIN>.com
  ingressClassName: nginx
  ingressAnnotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    
  grpcEnableTLS: true
  grpcIngressClassName: nginx
  grpcIngressAnnotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    
  env:
    - name: CORS_REST_ALLOWED_ORIGINS
      value: "http://keycloak.<YOUR-DOMAIN>.com"
    - name: CORS_REST_ALLOW_CREDENTIALS
      value: "true"

keycloak:
  enabled: true
  install: false
  url: keycloak.<YOUR-DOMAIN>.com
  privateUrl: http://keycloak.keycloak.svc.cluster.local:80
  realm: microcks
  client:
    id: <YOUR-CLIENT-ID>
    secret: <CLIENT-SECRET>

mongodb:
  install: false
  # Connect to the internal MongoDB service we deployed earlier
  uri: mongodb://mongodb.microcks.svc.cluster.local:27017/microcks
  uriParameters: "?authSource=admin"
  database: microcks
  secretRef:
    secret: microcks-mongodb-connection
    usernameKey: username
    passwordKey: password

ingress:
  enabled: true
  tls: false
```

### 6.3 Deploy Microcks
Deploy the official Microcks Helm chart using your configuration file:

```sh
helm repo add microcks https://microcks.io/helm/
helm install microcks microcks/microcks \
    -n microcks \
    -f microcks.yaml
```

### 6.4 Get Ingress IP and Access URL
Retrieve the external IP of your Vultr Load Balancer:

```sh
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

Point your DNS records (`microcks.<YOUR-DOMAIN>.com` and `keycloak.<YOUR-DOMAIN>.com`) to this `EXTERNAL-IP`.

Microcks will be available at: `http://microcks.<YOUR-DOMAIN>.com`

🎉 **Congratulations!** You have successfully deployed a production-grade Microcks instance on Vultr Cloud!
