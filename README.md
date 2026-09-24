# Kubernetes WordPress DevOps Project

## Overview

This project demonstrates the deployment and operation of a WordPress application on Kubernetes using **MicroK8s** on a **Debian 12 virtual machine**.

The environment includes:

* MicroK8s Kubernetes cluster
* Traefik Ingress Controller
* WordPress deployed using Helm
* PostgreSQL deployed in Kubernetes
* Persistent storage using PVCs
* Kubernetes Secrets for database credentials
* Kubernetes ConfigMaps
* PostgreSQL backup using Kubernetes CronJob
* TLS termination
* Nginx reverse proxy on the Debian VM
* Kubernetes NetworkPolicy
* Resource requests and limits
* Kubernetes health probes
* Security contexts
* Rolling updates
* Custom WordPress Docker image

---

# 1. Architecture

The infrastructure consists of a Debian 12 virtual machine running MicroK8s.

Traffic follows this path:

```text
Client
  |
  | HTTPS :443
  v
Nginx
  |
  | HTTP
  v
Traefik Ingress Controller
  |
  | Kubernetes Ingress
  v
WordPress Service
  |
  v
WordPress Pods
  |
  | PostgreSQL connection
  v
PostgreSQL Service
  |
  v
PostgreSQL StatefulSet
  |
  v
PersistentVolumeClaim
```

### Main components

| Component               | Technology                |
| ----------------------- | ------------------------- |
| Operating System        | Debian 12                 |
| Container orchestration | MicroK8s                  |
| Ingress Controller      | Traefik                   |
| Web application         | WordPress                 |
| Database                | PostgreSQL                |
| Reverse Proxy           | Nginx                     |
| Storage                 | Kubernetes PVC            |
| TLS                     | Kubernetes Secret / Nginx |
| Package management      | Helm                      |
| Container runtime       | containerd                |

**Screenshot:**
`[Insert architecture / network diagram here]`

---

# 2. Environment

The project was implemented on a Debian 12 virtual machine accessed through SSH.

Check the operating system:

```bash
cat /etc/os-release
```

Check the Kubernetes node:

```bash
microk8s kubectl get nodes -o wide
```

Check all Kubernetes workloads:

```bash
microk8s kubectl get pods -A
```
---

# 3. MicroK8s

MicroK8s was used as the Kubernetes distribution.

Required addons:

* DNS
* Ingress
* Metrics Server
* Storage

Check enabled addons:

```bash
microk8s status
```

Check Kubernetes nodes:

```bash
microk8s kubectl get nodes
```

Check all pods:

```bash
microk8s kubectl get pods -A
```

Example expected result:

```text
NAME       STATUS   ROLES    AGE
debian12   Ready    <none>   ...
```
---

# 4. PostgreSQL

PostgreSQL is deployed in the `database` namespace.

Check the namespace:

```bash
microk8s kubectl get all -n database
```

The database consists of:

* StatefulSet
* PostgreSQL Pod
* ClusterIP Service
* PersistentVolumeClaim
* Kubernetes Secret

Check StatefulSet:

```bash
microk8s kubectl get statefulset -n database
```

Check PostgreSQL Pod:

```bash
microk8s kubectl get pods -n database
```

Check Service:

```bash
microk8s kubectl get svc -n database
```

Expected service:

```text
postgres   ClusterIP   ...   5432/TCP
```

Check PVC:

```bash
microk8s kubectl get pvc -n database
```

---

# 5. PostgreSQL credentials

Database credentials are stored using a Kubernetes Secret.

Check the Secret:

```bash
microk8s kubectl get secret -n database
```

The passwords are not stored directly in the Kubernetes YAML manifests.

Inspect Secret keys without displaying the decoded passwords:

```bash
microk8s kubectl get secret postgres-credentials \
  -n database \
  -o jsonpath='{.data}' | \
  sed 's/,/\n/g' | \
  sed 's/:[^,}]*/: [REDACTED]/g'
```

The application uses the Secret instead of hard-coded credentials.

---

# 6. PostgreSQL database for WordPress

The WordPress database is hosted by PostgreSQL.

The Kubernetes service DNS name is:

```text
postgres.database.svc.cluster.local
```

Port:

```text
5432
```

The WordPress database configuration uses:

```text
WORDPRESS_DB_HOST=postgres.database.svc.cluster.local:5432
```

---

# 7. PostgreSQL connection test

A PostgreSQL client can be used to verify the connection.

For example, create a temporary PostgreSQL client:

```bash
microk8s kubectl run psql-test \
  -n database \
  --image=postgres:16 \
  --rm -it \
  --restart=Never \
  -- \
  psql -h postgres \
  -U postgres \
  -d postgres
```

After entering the password, run:

```sql
SELECT version();
```

Successful output confirms that the PostgreSQL service is reachable.

**Screenshot:**
`[Insert screenshot showing successful PostgreSQL connection and SELECT version() here]`

---

# 8. PostgreSQL backup

PostgreSQL backups are automated using a Kubernetes CronJob.

Check the CronJob:

```bash
microk8s kubectl get cronjob -n database
```

Check backup Jobs:

```bash
microk8s kubectl get jobs -n database
```

Check backup storage:

```bash
microk8s kubectl get pvc -n database
```

The backup process uses:

```text
pg_dump
```

and stores the resulting backup files on persistent storage.

---

# 9. Backup verification

List backup files inside the backup environment:

```bash
microk8s kubectl exec -n database <backup-pod> -- ls -lh /backups
```

Alternatively, inspect the completed backup Job:

```bash
microk8s kubectl describe job <job-name> -n database
```

View logs:

```bash
microk8s kubectl logs job/<job-name> -n database
```

**Screenshot:**
`[Insert screenshot showing CronJob and completed PostgreSQL backup Job here]`

---

# 10. PostgreSQL restore

A PostgreSQL backup can be restored using `psql` or the appropriate PostgreSQL restore command depending on the backup format.

Example:

```bash
psql -h postgres \
  -U postgres \
  -d wordpress_db \
  < backup.sql
```

After restoration, verify the database:

```sql
\dt
```

or:

```sql
SELECT current_database();
```
---

# 11. WordPress deployment

WordPress is deployed in the `wordpress` namespace.

Check the namespace:

```bash
microk8s kubectl get all -n wordpress
```

Check WordPress Pods:

```bash
microk8s kubectl get pods -n wordpress
```

Expected state:

```text
wordpress-...   1/1   Running
wordpress-...   1/1   Running
```

Two replicas are used to demonstrate Kubernetes deployment and rolling update functionality.

---

# 12. WordPress Service

Check the WordPress Service:

```bash
microk8s kubectl get svc -n wordpress
```

The WordPress application is exposed internally through a Kubernetes `ClusterIP` Service.

Example:

```text
wordpress-helm-wordpress   ClusterIP   ...   80/TCP
```

---

# 13. WordPress Persistent Storage

WordPress uses a PersistentVolumeClaim for persistent application data.

Check PVCs:

```bash
microk8s kubectl get pvc -n wordpress
```

Describe the PVC:

```bash
microk8s kubectl describe pvc wordpress-data -n wordpress
```

Persistent storage ensures that application data is not lost when a WordPress Pod is recreated.

---

# 14. WordPress database configuration

The WordPress deployment receives its database configuration from Kubernetes resources.

Database host:

```text
postgres.database.svc.cluster.local:5432
```

Database credentials are provided through a Kubernetes Secret.

Check the Deployment configuration:

```bash
microk8s kubectl describe deployment wordpress-helm-wordpress -n wordpress
```

---

# 15. Helm

WordPress was deployed using Helm.

List Helm releases:

```bash
microk8s helm3 list -A
```

Example:

```text
wordpress-helm
```

Show the configured values:

```bash
microk8s helm3 get values wordpress-helm -n wordpress -a
```

For the Traefik installation:

```bash
microk8s helm3 get values traefik -n ingress -a
```
---

# 16. Custom WordPress image

The WordPress deployment uses a custom image:

```text
wordpress-custom:1.2
```

Check the image used by the Pods:

```bash
microk8s kubectl get pods -n wordpress -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec.containers[*].image}{"\n"}{end}'
```

The custom image contains the PostgreSQL compatibility components required by the application.

---

# 17. pg4wp configuration

The custom WordPress image contains the PostgreSQL WordPress integration.

The relevant files are:

```text
wp-content/pg4wp
wp-content/db.php
```

They are copied into the WordPress persistent volume by the init container.

Check the init container:

```bash
microk8s kubectl describe pod -n wordpress <wordpress-pod>
```

The init container performs operations similar to:

```text
mkdir -p /work/wp-content
cp -a /usr/src/wordpress/wp-content/pg4wp /work/wp-content/pg4wp
cp /usr/src/wordpress/wp-content/db.php /work/wp-content/db.php
```

---

# 18. WordPress readiness and liveness

The WordPress Deployment uses Kubernetes health checks.

Check the Deployment:

```bash
microk8s kubectl describe deployment wordpress-helm-wordpress -n wordpress
```

The output can be used to demonstrate:

* Readiness probe
* Liveness probe
* Resource requests
* Resource limits
* Rolling update strategy

---

# 19. Resource limits

The WordPress container has configured resource requests and limits.

Example:

```text
Requests:
  CPU:    100m
  Memory: 256Mi

Limits:
  CPU:    500m
  Memory: 512Mi
```

This prevents the application from consuming unlimited node resources.

---

# 20. Rolling updates

Check the Deployment strategy:

```bash
microk8s kubectl get deployment wordpress-helm-wordpress \
  -n wordpress \
  -o yaml | grep -A8 strategy
```

The Deployment uses:

```text
RollingUpdate
```

This allows Pods to be replaced gradually during an application update.

---

# 21. Ingress

Traefik is used as the Kubernetes Ingress Controller.

Check Traefik:

```bash
microk8s kubectl get pods -n ingress
```

Check the Traefik Service:

```bash
microk8s kubectl get svc -n ingress
```

The service exposes:

```text
HTTP  : 80
HTTPS : 443
```

Check the WordPress Ingress:

```bash
microk8s kubectl get ingress -n wordpress
```

Detailed configuration:

```bash
microk8s kubectl describe ingress wordpress-helm-wordpress -n wordpress
```

The configured hostname is:

```text
wordpress.local
```

---

# 22. TLS

TLS is configured for:

```text
wordpress.local
```

The Ingress uses the Kubernetes TLS Secret:

```text
wordpress-tls
```

Check it:

```bash
microk8s kubectl get secret wordpress-tls -n wordpress
```

The Nginx reverse proxy also uses the TLS certificate:

```text
/etc/nginx/ssl/wordpress.local.crt
/etc/nginx/ssl/wordpress.local.key
```

---

# 23. HTTP and HTTPS testing

The root WordPress URL currently redirects to the WordPress installation page.

Therefore:

```text
HTTP / HTTPS root → HTTP/HTTPS 302 → /wp-admin/install.php
```

This is a WordPress application redirect and does not necessarily indicate an Nginx or Kubernetes error.

Test the final HTTPS page:

```bash
curl -k -I https://wordpress.local/wp-admin/install.php
```

Expected result:

```text
HTTP/2 200
```

Example:

```text
HTTP/2 200
content-type: text/html; charset=utf-8
```

---

# 24. Nginx reverse proxy

Nginx runs directly on the Debian VM.

Check the service:

```bash
sudo systemctl status nginx
```

Check the configuration:

```bash
sudo nginx -t
```

The Nginx configuration contains two server blocks:

```text
HTTP :80
HTTPS :443
```

HTTP traffic is redirected to HTTPS:

```nginx
return 301 https://wordpress.local$request_uri;
```

HTTPS traffic is proxied to the Traefik NodePort:

```nginx
proxy_pass http://127.0.0.1:31064;
```

---

# 25. Nginx configuration verification

Display the active configuration:

```bash
sudo nginx -T | grep -n -A20 -B5 "wordpress.local"
```

The configuration should contain:

```nginx
server {
    listen 80;
    server_name wordpress.local;

    location / {
        return 301 https://wordpress.local$request_uri;
    }
}
```

and:

```nginx
server {
    listen 443 ssl;
    server_name wordpress.local;

    location / {
        proxy_pass http://127.0.0.1:31064;
    }
}
```

---

# 26. Security

Security controls implemented in the Kubernetes environment include:

* Kubernetes Secrets for credentials
* No plaintext database passwords in application manifests
* Non-root container configuration where supported
* SecurityContext
* Dropped Linux capabilities
* `allowPrivilegeEscalation: false`
* Persistent storage
* TLS
* NetworkPolicy
* Kubernetes resource limits
* Kubernetes health probes
* Non-privileged workloads

---

# 27. SecurityContext verification

Check the WordPress Pod:

```bash
microk8s kubectl get pod -n wordpress <wordpress-pod> -o yaml
```

Check security-related configuration:

```bash
microk8s kubectl get pod -n wordpress <wordpress-pod> -o yaml | grep -A15 -B5 securityContext
```

For Traefik, the Helm configuration includes:

```text
runAsNonRoot: true
allowPrivilegeEscalation: false
capabilities:
  drop:
    - ALL
readOnlyRootFilesystem: true
```

---

# 28. NetworkPolicy

Network policies are used to restrict communication between namespaces.

Check NetworkPolicies:

```bash
microk8s kubectl get networkpolicy -A
```

Describe a policy:

```bash
microk8s kubectl describe networkpolicy <policy-name> -n database
```

The intended architecture allows the application namespace to communicate with PostgreSQL while restricting unnecessary database access.

---

# 29. Final Kubernetes status

Before completing the deployment, verify all major components.

### Nodes

```bash
microk8s kubectl get nodes
```

### All Pods

```bash
microk8s kubectl get pods -A
```

### Services

```bash
microk8s kubectl get svc -A
```

### Persistent volumes

```bash
microk8s kubectl get pv
```

### PersistentVolumeClaims

```bash
microk8s kubectl get pvc -A
```

### Ingress

```bash
microk8s kubectl get ingress -A
```

### NetworkPolicies

```bash
microk8s kubectl get networkpolicy -A
```

### CronJobs

```bash
microk8s kubectl get cronjob -A
```
---

# 30. Troubleshooting

## Check WordPress Pods

```bash
microk8s kubectl get pods -n wordpress
```

## Describe a WordPress Pod

```bash
microk8s kubectl describe pod -n wordpress <pod-name>
```

## Check WordPress logs

```bash
microk8s kubectl logs -n wordpress <pod-name> -c wordpress
```

## Check init container logs

```bash
microk8s kubectl logs -n wordpress <pod-name> -c install-pg4wp
```

## Check PostgreSQL

```bash
microk8s kubectl get pods -n database
```

## PostgreSQL logs

```bash
microk8s kubectl logs -n database <postgres-pod>
```

## Check Ingress

```bash
microk8s kubectl describe ingress wordpress-helm-wordpress -n wordpress
```

## Check Traefik

```bash
microk8s kubectl logs -n ingress <traefik-pod>
```

## Check Nginx

```bash
sudo nginx -t
sudo systemctl status nginx
sudo tail -50 /var/log/nginx/error.log
```

---

# 31. GitHub repository structure

Recommended repository structure:

```text
.
├── README.md
├── docker/
│   ├── Dockerfile
│   └── .dockerignore
├── kubernetes/
│   ├── database/
│   │   ├── namespace.yaml
│   │   ├── statefulset.yaml
│   │   ├── service.yaml
│   │   ├── pvc.yaml
│   │   ├── secret.yaml
│   │   ├── backup-cronjob.yaml
│   │   └── networkpolicy.yaml
│   │
│   └── wordpress/
│       ├── namespace.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── pvc.yaml
│       ├── ingress.yaml
│       ├── configmap.yaml
│       ├── secret.yaml
│       └── networkpolicy.yaml
│
├── helm/
│   └── wordpress/
│
└── docs/
    └── screenshots/
```

Sensitive credentials should not be committed to GitHub.

---

# 32. Useful verification commands

### Kubernetes

```bash
microk8s kubectl get nodes
microk8s kubectl get pods -A
microk8s kubectl get svc -A
microk8s kubectl get pvc -A
microk8s kubectl get ingress -A
```

### PostgreSQL

```bash
microk8s kubectl get statefulset -n database
microk8s kubectl get pvc -n database
microk8s kubectl get cronjob -n database
microk8s kubectl get jobs -n database
```

### WordPress

```bash
microk8s kubectl get deployment -n wordpress
microk8s kubectl get pods -n wordpress
microk8s kubectl get svc -n wordpress
microk8s kubectl get ingress -n wordpress
```

### Ingress

```bash
microk8s kubectl get pods -n ingress
microk8s kubectl get svc -n ingress
```

### Nginx

```bash
sudo nginx -t
sudo systemctl status nginx
```

### HTTPS

```bash
curl -k -I https://wordpress.local/wp-admin/install.php
```

Expected:

```text
HTTP/2 200
```

---

# 33. Conclusion

This project demonstrates a complete Kubernetes-based WordPress deployment on Debian 12 using MicroK8s.

The environment includes:

* Kubernetes orchestration
* PostgreSQL StatefulSet
* Persistent storage
* Automated database backups
* WordPress Deployment
* Kubernetes Services
* Traefik Ingress
* TLS
* Nginx reverse proxy
* Kubernetes Secrets
* ConfigMaps
* NetworkPolicy
* SecurityContext
* Resource management
* Health checks
* Rolling updates
* Helm-based deployment
* Custom WordPress container image

The configuration provides a reproducible foundation for deploying a stateful web application using Kubernetes while applying common DevOps and security practices.
