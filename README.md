# devops-aprentiship
# WordPress on Kubernetes

## 1. Project Overview

This project demonstrates deployment of a production-style WordPress application on a local Kubernetes cluster using MicroK8s.

The infrastructure includes:

* MicroK8s Kubernetes cluster
* WordPress
* MariaDB
* Persistent storage
* Kubernetes Secrets
* ClusterIP Services
* NGINX Ingress
* TLS/HTTPS
* Resource requests and limits
* Liveness and readiness probes
* Horizontal Pod Autoscaler
* PodDisruptionBudget
* Helm chart
* Scheduled database backups

The project is designed to demonstrate Kubernetes administration, containerisation, persistent storage, application availability and Helm-based deployment.

---

## 2. Architecture

The application follows this architecture:

```text
                         Client
                           |
                           | HTTPS
                           v
                    NGINX Ingress
                           |
                           v
                 WordPress ClusterIP
                           |
                           v
                 WordPress Deployment
                    /             \
                   /               \
             Pod 1                  Pod 2
                   \               /
                    \             /
                     Persistent
                       Storage
                           |
                           v
                       MariaDB
                           |
                           v
                    MariaDB PVC
```

### Main components

| Component            | Purpose                                                        |
| -------------------- | -------------------------------------------------------------- |
| WordPress Deployment | Runs the WordPress application                                 |
| MariaDB              | Stores WordPress data                                          |
| PVC                  | Provides persistent storage                                    |
| Service              | Provides internal Kubernetes networking                        |
| Ingress              | Provides HTTP/HTTPS access                                     |
| Secret               | Stores database credentials                                    |
| HPA                  | Scales WordPress based on CPU utilisation                      |
| PDB                  | Protects application availability during voluntary disruptions |
| Helm                 | Provides templated and repeatable deployment                   |
| CronJob              | Performs scheduled database backups                            |

---

## 3. Prerequisites

The following software is required:

* Debian 12
* MicroK8s
* kubectl / `microk8s kubectl`
* Helm
* Docker
* Git

Required MicroK8s addons:

```bash
microk8s enable dns
microk8s enable ingress
microk8s enable metrics-server
microk8s enable storage
```

Verify the cluster:

```bash
microk8s status
microk8s kubectl get nodes
```

The node should report:

```text
Ready
```

---

## 4. Deployment

Clone the repository:

```bash
git clone <REPOSITORY_URL>
cd <REPOSITORY_DIRECTORY>
```

Create the required namespaces:

```bash
microk8s kubectl apply -f manifest/namespace.yaml
```

Deploy the database:

```bash
microk8s kubectl apply -f manifest/mariadb.yaml
```

Verify MariaDB:

```bash
microk8s kubectl get pods -n wordpress
microk8s kubectl get pvc -n wordpress
microk8s kubectl get svc -n wordpress
```

Deploy WordPress:

```bash
microk8s kubectl apply -f manifest/wordpress.yaml
```

Deploy networking:

```bash
microk8s kubectl apply -f manifest/ingress.yaml
```

Verify:

```bash
microk8s kubectl get pods -n wordpress
microk8s kubectl get svc -n wordpress
microk8s kubectl get ingress -n wordpress
```

---

## 5. Configuration and Secrets

Database credentials are stored in Kubernetes Secrets rather than directly in YAML manifests.

Example:

```text
mariadb-credentials
```

The WordPress container receives its database configuration through environment variables.

The database host is:

```text
mariadb.wordpress.svc.cluster.local:3306
```

The database name, username and password are retrieved from Kubernetes Secret references.

This prevents credentials from being hard-coded into application manifests.

---

## 6. Persistent Storage

WordPress uses a PersistentVolumeClaim for `/var/www/html`.

MariaDB also uses persistent storage for the database files.

Persistent volumes ensure that application and database data are not lost when Pods are recreated.

Verify storage:

```bash
microk8s kubectl get pvc -A
```

---

## 7. Health Checks

The WordPress Deployment uses Kubernetes readiness and liveness probes.

Readiness probes prevent Kubernetes from sending traffic to an application that is not ready.

Liveness probes allow Kubernetes to restart a container that has become unhealthy.

Example:

```yaml
readinessProbe:
  httpGet:
    path: /wp-admin/install.php
    port: 80

livenessProbe:
  httpGet:
    path: /wp-admin/install.php
    port: 80
```

---

## 8. Resource Management

The WordPress container defines CPU and memory requests and limits.

Example:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

Requests allow Kubernetes to make scheduling decisions.

Limits prevent a container from consuming unlimited resources.

---

## 9. Horizontal Pod Autoscaler

HPA automatically adjusts the number of WordPress replicas based on CPU utilisation.

Current configuration:

```text
Minimum replicas: 2
Maximum replicas: 3
CPU target: 70%
```

Check HPA:

```bash
microk8s kubectl get hpa -n wordpress
```

Detailed information:

```bash
microk8s kubectl describe hpa -n wordpress
```

---

## 10. PodDisruptionBudget

A PodDisruptionBudget is configured to maintain application availability during voluntary disruptions.

Example:

```yaml
minAvailable: 1
```

Check the PDB:

```bash
microk8s kubectl get pdb -n wordpress
```

---

## 11. Helm Deployment

The project also contains a Helm chart:

```text
helm/wordpress/
├── Chart.yaml
├── values.yaml
└── templates/
```

Validate the chart:

```bash
helm lint helm/wordpress
```

Render templates:

```bash
helm template wordpress-helm helm/wordpress -n wordpress
```

Install:

```bash
helm install wordpress-helm helm/wordpress -n wordpress
```

Upgrade:

```bash
helm upgrade wordpress-helm helm/wordpress -n wordpress
```

Check the release:

```bash
helm status wordpress-helm -n wordpress
```

---

## 12. Access to WordPress

WordPress is exposed through the Kubernetes Ingress.

The configured hostname is:

```text
wordpress.local
```

For local testing, the hostname must resolve to the Kubernetes node.

Example `/etc/hosts` entry:

```text
<VM_IP> wordpress.local
```

The application is then accessed using:

```text
https://wordpress.local
```

TLS is configured using the Kubernetes TLS secret:

```text
wordpress-tls
```

---

## 13. Verification

Useful verification commands:

```bash
microk8s kubectl get nodes
```

```bash
microk8s kubectl get pods -n wordpress
```

```bash
microk8s kubectl get svc -n wordpress
```

```bash
microk8s kubectl get ingress -n wordpress
```

```bash
microk8s kubectl get pvc -n wordpress
```

```bash
microk8s kubectl get hpa -n wordpress
```

```bash
microk8s kubectl get pdb -n wordpress
```

```bash
helm list -n wordpress
```

---

## 14. Backup

The database backup is implemented using a Kubernetes CronJob.

The CronJob periodically creates a backup of the MariaDB database.

Verify the CronJob:

```bash
microk8s kubectl get cronjob -n wordpress
```

Check created Jobs:

```bash
microk8s kubectl get jobs -n wordpress
```

---

## 15. Screenshots

The following screenshots document the working deployment:

### Kubernetes cluster

![Kubernetes cluster](screenshots/01-cluster.png)

### Running Pods

![Running Pods](screenshots/02-pods.png)

### WordPress

![WordPress](screenshots/03-wordpress.png)

### Ingress and HTTPS

![Ingress](screenshots/04-ingress.png)

### HPA

![HPA](screenshots/05-hpa.png)

### Helm

![Helm](screenshots/06-helm.png)

---

## 16. Design Decisions

### Kubernetes

Kubernetes was selected because it provides container orchestration, service discovery, health checks, scaling and declarative infrastructure management.

### MicroK8s

MicroK8s was used because it provides a lightweight Kubernetes distribution suitable for a single-node development and testing environment.

### MariaDB

MariaDB is used as the relational database backend required by WordPress.

### PersistentVolumeClaims

Persistent storage is required because WordPress and MariaDB must preserve data across Pod restarts.

### Kubernetes Secrets

Database credentials are stored in Secrets rather than being hard-coded into Deployment manifests.

### Ingress

Ingress provides a single HTTP/HTTPS entry point for WordPress and enables TLS termination.

### HPA

HPA allows the WordPress application to scale horizontally when CPU utilisation increases.

### PDB

PDB reduces the risk of voluntary disruptions removing all available WordPress replicas simultaneously.

### Helm

Helm provides parameterisation and repeatable deployments and separates configuration from Kubernetes templates.

---

## 17. Repository Structure

```text
.
├── README.md
├── Dockerfile
├── manifest/
│   ├── namespace.yaml
│   ├── mariadb.yaml
│   ├── wordpress.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   └── ...
├── helm/
│   └── wordpress/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
└── screenshots/
```

All Kubernetes YAML manifests and the Helm chart are stored in the Git repository for reproducible deployment.

---

## 18. Additional Improvements

The following improvements can be implemented as additional Kubernetes/DevOps features:

* CI/CD using GitHub Actions
* Automated Docker image build
* Automated Helm deployment
* Zero-downtime application updates
* Automated WordPress updates
* Database migration strategy
* Improved backup and restore procedure
* Production-grade monitoring
* NetworkPolicies
* Pod security hardening
