# 🚀 CNPG + cert-manager Example Setup

A complete walkthrough for setting up a PostgreSQL cluster (TLS and mTLS configuration) using [CloudNativePG](https://cloudnative-pg.io) and [cert-manager](https://cert-manager.io) in a local [Kind](https://kind.sigs.k8s.io) Kubernetes cluster.

---

## ✅ Prerequisites

Before you begin, make sure the following tools are installed on your machine:

- 🛠️ [**Helm**](https://helm.sh/docs/intro/install/) – Kubernetes package manager
- 🐳 [**Kind**](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) – Run Kubernetes in Docker
- 📦 [**kubectl**](https://kubernetes.io/docs/tasks/tools/#kubectl) – Kubernetes CLI
- 🔌 [**kubectl-cnpg Plugin**](https://cloudnative-pg.io/documentation/1.25/kubectl-plugin/) – CloudNativePG `kubectl` extension

---

## 📖 Step-by-Step Setup Guide

### 🧱 1. Create a Kind Cluster

Spin up a local Kubernetes cluster using Kind:

```bash
kind create cluster
```

---

### 🔄 2. Set the Kubernetes Context

Ensure your kubectl context is pointing to the new Kind cluster:

```bash
kubectl config use-context kind-kind
```

---

### 🔐 3. Install cert-manager with Helm

Add the `jetstack` Helm repository and install `cert-manager`:

```bash
helm repo add jetstack https://charts.jetstack.io --force-update

helm upgrade --install cert-manager jetstack/cert-manager \
  --create-namespace \
  --namespace cert-manager \
  --set crds.enabled=true \
  --set enableCertificateOwnerRef=true \
  --set global.leaderElection.namespace=cert-manager \
  --version v1.17.2

kubectl --namespace cert-manager wait --for=condition=ready pod --selector app.kubernetes.io/component=controller,app.kubernetes.io/instance=cert-manager --timeout=300s
```

📝 This setup enables certificate resource definitions and sets up leader election in the correct namespace.

---

### 🧾 4. Deploy a ClusterIssuer

Apply your cert-manager configuration (e.g., a `ClusterIssuer`) from the provided manifests:

```bash
kubectl apply -f cert-manager/
```

---

### 🐘 5. Install CloudNativePG

Deploy CloudNativePG operator to manage your PostgreSQL clusters:

```bash
kubectl apply --server-side -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.25/releases/cnpg-1.25.1.yaml

kubectl --namespace cnpg-system wait --for=condition=ready pod --selector app.kubernetes.io/name=cloudnative-pg --timeout=300s
```

---

### 📂 6. Create a Namespace for Your PostgreSQL Cluster

Create the Kubernetes namespace that your PostgreSQL cluster will run in:

```bash
kubectl create namespace dts-test
```

📁 This isolates your PostgreSQL resources under the `dts-test` namespace.

---

### 📦 7. Deploy a PostgreSQL Cluster

You can choose to deploy your cluster using either **Mutual TLS (mTLS)** or standard **TLS**, depending on your security requirements.

> ⚠️ The database backup is intentionally not configured.

#### 🔒 7.1 mTLS (Mutual TLS)

To deploy a PostgreSQL cluster that uses **mutual TLS authentication** (client and server certificates):

```bash
kubectl apply -f postgresql-cluster/mtls/
```

🛡️ This setup enforces certificate validation on both the client and server sides.

#### 🔐 7.2 TLS (Server-only TLS)

To deploy a PostgreSQL cluster that uses **TLS with server-side certificate only**:

```bash
kubectl apply -f postgresql-cluster/tls/
```

🔐 This setup ensures encrypted connections to PostgreSQL, while client certs are not required.

---

### 🔍 8. Check PostgreSQL Cluster Status

Monitor the status of your deployed PostgreSQL cluster using the cnpg plugin:

```bash
kubectl cnpg -n dts-test status dts-postgres --verbose
```

Wait until the cluster is in healthy state:

```bash
Cluster Summary
Name                 dts-test/dts-postgres
(...)
Status:              Cluster in healthy state
Instances:           3
Ready instances:     3
```

📊 This gives you detailed information about the cluster health, instance status, and more.

---

### 🧪 9. Test Connection of the R/W instance

You can test connectivity to your PostgreSQL cluster by deploying a `psql-client` pod with the correct configuration.

We're using the CloudNativePG PostgreSQL server image because it includes a [psql client](https://www.postgresql.org/docs/17/app-psql.html) that's compatible with the server. However, feel free to use any other container image that provides a compatible `psql` client.

#### 🔒 9.1 mTLS (Mutual TLS)

Deploy a `psql-client` pod configured for mutual TLS:

```bash
kubectl apply -f postgresql-client/mtls/
kubectl --namespace dts-test wait --for=condition=ready pod --selector app.kubernetes.io/name=psql-client --timeout=300s
```

Once it's running, connect using:

```bash
kubectl --namespace dts-test exec -it psql-client -c client -- psql
```

🔐 Ensure the pod has the correct client certificate and CA to trust the server.

---

#### 🔐 9.2 TLS (Server-only TLS)

Deploy a `psql-client` pod configured for standard TLS:

```bash
kubectl apply -f postgresql-client/tls/
kubectl --namespace dts-test wait --for=condition=ready pod --selector app.kubernetes.io/name=psql-client --timeout=300s
```

Once it's running, connect using:

```bash
kubectl --namespace dts-test exec -it psql-client -c client -- psql
```

💡 This approach only requires the server’s certificate to be trusted by the client.

---

### 🧪 10. Test Connection of the PgBouncer

Use the `psql` client inside a running pod to connect to PgBouncer:

```bash
kubectl --namespace dts-test wait --for=condition=ready pod --selector app.kubernetes.io/name=psql-pgbouncer-client --timeout=300s
kubectl --namespace dts-test exec -it psql-pgbouncer-client -c client -- psql
```

> ⚠️ **Note:**  The CloudNativePG operator currently does not populate the connection pooler address in the generated Kubernetes secret. Most probably the maintainers will [redesign pooler management](https://github.com/cloudnative-pg/cloudnative-pg/issues/6869#issuecomment-2664901025).

---

## 🧹 Cleanup

Once you're done with testing, you can remove everything by deleting the Kind cluster:

```bash
kind delete cluster
```

🧼 This will clean up all Kubernetes resources and free up system resources.

---

## 🎉 You're Done!

Your local PostgreSQL cluster with cert-manager integration is now up and running! 🥳
