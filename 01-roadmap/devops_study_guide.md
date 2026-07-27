# DevOps, Infrastructure, & Continuous Delivery Study Guide

This guide covers cloud networking, container optimization, Kubernetes orchestration, Infrastructure as Code (Terraform), GitOps CD (ArgoCD), and CI pipelines (Jenkins, GitHub Actions, GitLab CI/CD).

---

## 1. Cloud Networking & Infrastructure (VPC & Subnets)

Relational databases and backend microservices must run inside isolated network boundaries:

### Virtual Private Cloud (VPC) Architecture

```
[ Internet ]
     │
     ▼
[ Internet Gateway ]
     │
     ▼
[ Public Subnet ] (Contains Load Balancers, Bastion/VPN Hosts)
     │
     ▼
[ NAT Gateway ] (Allows private instances to fetch internet updates)
     │
     ▼
[ Private Subnet ] (Contains App Services, Kubernetes Worker Nodes)
     │
     ▼
[ Database Subnet ] (Contains PostgreSQL / Cloud SQL - No Internet Routes)
```

* **Public Subnet:** Direct routing to the **Internet Gateway (IGW)**. Holds internet-facing resources like Application Load Balancers (ALBs) or Bastion Hosts.
* **Private Subnet:** Routing table redirects outbound traffic to a **NAT Gateway** located in a public subnet. Prevents direct inbound internet connections.
* **Bastion Host (Jump Box):** A secure VM located in the public subnet. Engineers SSH into the Bastion (using key pairs) to access backend instances in the private subnet.

---

## 2. Containerization: Docker Optimization

To build secure, lightweight images for production environments:

### Multi-Stage Build & Hardening Blueprint (Node.js Example)
```dockerfile
# Stage 1: Build & Dependencies
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Stage 2: Runtime Production
FROM node:20-alpine
WORKDIR /app
# Run as non-root user for security hardening
USER node
COPY --chown=node:node --from=builder /app/node_modules ./node_modules
COPY --chown=node:node --from=builder /app/dist ./dist
COPY --chown=node:node --from=builder /app/package.json ./package.json

EXPOSE 3000
CMD ["node", "dist/main.js"]
```

### Docker Best Practices
1. **Use Multi-Stage Builds:** Separates the compiler tools (npm build, JDK compile) from the lightweight runtime image (node, JRE), reducing image size from 1GB+ to under 150MB.
2. **Minimize Layers:** Group commands (`RUN apt-get update && apt-get install -y ... && rm -rf /var/lib/apt/lists/*`) to reduce the final image layers.
3. **Never Run as Root:** Enforce a non-root user (e.g. `USER node` or custom UID) to prevent host access exploits if the container is compromised.
4. **Use Explicit Tags:** Avoid `latest`. Use explicit versions (e.g. `node:20.11-alpine`) to ensure builds are reproducible.

---

## 3. Container Orchestration: Kubernetes (K8s)

Kubernetes orchestrates container deployment, scaling, and networking.

### Control Plane vs. Worker Nodes
* **Control Plane (Master Node):**
  * `kube-apiserver`: The entrypoint gatekeeper for administrative commands.
  * `etcd`: Strongly consistent distributed key-value store holding the cluster state.
  * `kube-scheduler`: Assigns newly created Pods to Worker Nodes based on resource availability.
  * `kube-controller-manager`: Runs controller loops (e.g. Node Controller, ReplicaSet Controller).
* **Worker Node:**
  * `kubelet`: Agent running on each node verifying containers match PodSpecs.
  * `kube-proxy`: Directs network traffic to correct pods (IPVS / iptables).
  * Container Runtime (containerd / Docker).

### Core Resources
* **Pods:** The smallest deployable units containing one or more containers sharing a network IP.
* **Deployments:** Manages stateless pods, enabling rolling updates and rollbacks.
* **StatefulSets:** Manages stateful pods (databases, Kafka brokers), providing stable network identifiers and persistent storage bindings.
* **Ingress:** Manages external HTTP routing into the cluster (acting as a reverse proxy).
* **Autoscaling (HPA & KEDA):**
  * **HPA (Horizontal Pod Autoscaler):** Scales pod replicas based on CPU/Memory usage.
  * **KEDA (Kubernetes Event-driven Autoscaling):** Scales pods based on external metrics (e.g. Kafka lag size, Redis queue length).

---

## 4. Package Management: Helm

Helm is a package manager for Kubernetes resources.

* **Charts:** Packages of pre-configured K8s resource templates (Deployments, Services, ConfigMaps).
* **Values.yaml:** Separation of configuration from structure. Defines environment-specific variables (e.g., replica count, db connection strings) passed to the templates at release time.
* **Releases:** Instances of a chart running inside a specific namespace in a Kubernetes cluster.

---

## 5. Infrastructure as Code (IaC): Terraform

Terraform manages cloud resources declaratively.

### Core Architecture
* **State File (`terraform.tfstate`):** Tracks physical resource IDs mapped to configuration templates.
* **State Locking:** Essential when team members run Terraform concurrently. Using remote backends (like AWS S3 + DynamoDB or GCP Cloud Storage) locks the state to prevent corruption.
* **Drift:** Discrepancy between actual cloud resource configuration and the local state file. Running `terraform plan` detects drift and proposes changes to realign the infrastructure.

### Terraform Syntax Example (PostgreSQL Instance)
```hcl
resource "google_sql_database_instance" "postgres" {
  name             = "production-db-instance"
  database_version = "POSTGRES_15"
  region           = "asia-southeast1"

  settings {
    tier = "db-f1-micro"
    ip_configuration {
      ipv4_enabled    = false # Disable public IP access
      private_network = var.vpc_id # Route inside private VPC
    }
  }
}
```

---

## 6. GitOps Continuous Delivery: ArgoCD

ArgoCD is a declarative GitOps continuous delivery tool for Kubernetes.

```
[ Git Repo (manifests) ] ──► [ ArgoCD Controller ] ──► Reconciles state in ──► [ K8s Cluster ]
```

* **Git as the Single Source of Truth:** Kubernetes manifests (yaml files, Helm values) are stored in a Git repository.
* **Reconciliation Loop:** ArgoCD continuously compares the target Git repository state (desired state) against the active Kubernetes cluster state (live state).
* **Sync States:**
  * **Synced:** Live cluster configuration matches the Git repository.
  * **OutOfSync:** A drift is detected. ArgoCD can auto-sync or alert the team to reconcile.

---

## 7. Continuous Integration: CI Pipelines

### Pipeline Comparison

```
+------------------------+------------------------+------------------------+
| Jenkins                | GitHub Actions         | GitLab CI/CD           |
+------------------------+------------------------+------------------------+
| - Self-hosted server.  | - Cloud-hosted/Runners.| - Single application   |
| - Groovy Scripting.    | - YAML configuration.  |   covering dev lifecycle|
| - Highly customizable  | - Deep GitHub integration| - YAML configuration.  |
|   via plugins.         | - Marketplace Actions. | - Shared runners.      |
+------------------------+------------------------+------------------------+
```

### GitHub Actions Pipeline Example
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Leverage dependency caching to speed up runs
      - name: Cache Node Modules
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm ci
      - run: npm run test
      - run: npm run build
```

---

### Questions & Answers: DevOps & Infrastructure

#### Q1: What is the difference between a rolling update and a blue/green deployment strategy in Kubernetes?
**Answer:**
> "1. **Rolling Update (K8s Default):** Gradually replaces old pods with new pods. A new pod is created, waits to pass its readiness probe, and then routes traffic while one old pod is deleted. 
> *Advantage:* Low resource overhead.
> *Disadvantage:* Two different versions of the application run concurrently, requiring backend APIs to support backward-compatibility.
> 2. **Blue/Green Deployment:** Spawns a complete duplicate environment containing the new version (Green) alongside the active production environment (Blue). Once the Green environment passes testing, the Load Balancer/Service router switches all traffic to Green.
> *Advantage:* Rapid, clean rollback if bugs are detected.
> *Disadvantage:* Requires double the infrastructure resources during the deployment window."

#### Q2: Explain why you should configure `readinessProbe` and `livenessProbe` in a Kubernetes pod manifest.
**Answer:**
> "1. **`readinessProbe`:** Tells Kubernetes when the pod is ready to accept user network traffic. If a Java/Spring Boot pod takes 40 seconds to start, the readiness probe fails until the endpoint `/actuator/health` returns 200. This prevents routing traffic to starting pods.
> 2. **`livenessProbe`:** Tells Kubernetes when the pod is alive. If the application deadlocks or runs out of memory, the liveness probe fails. Kubernetes automatically terminates the deadlocked container and spawns a new instance to recover."

---
DevOps & Infrastructure Study Guide
