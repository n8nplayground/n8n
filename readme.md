# n8n + PostgreSQL Helm Deployment on Minikube (Windows + WSL)

## 💡 Overview

This setup runs `n8n` and `PostgreSQL` in a Minikube cluster inside **WSL (Windows Subsystem for Linux)** with **persistent volumes**. You will:

* Configure storage inside WSL (not Windows NTFS)
* Run Minikube with custom resources
* Mount hostPath volumes
* Create custom Helm chart for both containers
* Fix permission issues on PostgreSQL volume

---

## 🧱 Folder Structure

Place everything under:

```bash
/home/ahmetserdargeze/minikube-storage
├── n8n
└── postgres
```

Create folders with:

```bash
mkdir -p ~/minikube-storage/n8n
mkdir -p ~/minikube-storage/postgres
```

Set proper permissions for PostgreSQL:

```bash
sudo chown -R 999:999 ~/minikube-storage/postgres
chmod -R 700 ~/minikube-storage/postgres
```

---

## 🚀 Minikube Setup

```bash
minikube start \
  --cpus=6 \
  --memory=12288 \
  --driver=docker \
  --mount \
  --mount-string="/home/ahmetserdargeze/minikube-storage:/mnt/data"
```

> Ensure you’re in WSL (Ubuntu) and not using NTFS paths.

---

## 📁 Helm Chart Setup

Your Helm chart structure (e.g., `n8n-helm-chart/`) should look like:

```bash
n8n-helm-chart/
├── charts
├── templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── pvc.yaml
│   └── pv.yaml
├── values.yaml
└── Chart.yaml
```

### 📄 values.yaml

```yaml
n8n:
  image: n8nio/n8n
  tag: latest
  port: 5678
  mountPath: /home/node/.n8n
  volumePath: /mnt/data/n8n

postgres:
  image: postgres
  tag: latest
  port: 5432
  database: n8ndb
  host: postgres
  username: n8n
  password: n8npass
  mountPath: /var/lib/postgresql/data
  volumePath: /mnt/data/postgres
```

### 🧩 Install with Helm

Run the following command inside the chart directory:

```bash
helm install n8n-stack . -f values.yaml
```

To upgrade:

```bash
helm upgrade n8n-stack . -f values.yaml
```

To uninstall:

```bash
helm uninstall n8n-stack
```

---

## 🐘 PostgreSQL Issues

### ❌ `Operation not permitted` or `invalid permissions`

```bash
initdb: error: could not change permissions of directory "/var/lib/postgresql/data"
FATAL: data directory has invalid permissions
```

✅ Solution:

```bash
sudo chown -R 999:999 ~/minikube-storage/postgres
chmod -R 700 ~/minikube-storage/postgres
```

### ❌ `role "n8n" does not exist`

PostgreSQL doesn’t auto-create roles. Ensure the Helm chart sets `POSTGRES_USER=n8n` and PVC is clean.

To re-init:

```bash
kubectl delete pvc postgres-pvc
kubectl delete pv postgres-pv
rm -rf ~/minikube-storage/postgres/*
```

---

## 🌐 Access n8n UI

Expose via `NodePort` or port-forward:

```bash
kubectl port-forward svc/n8n-service 5678:5678
```

Open: [http://localhost:5678](http://localhost:5678)

You can also enable port-forwarding with Docker for GUI access via Minikube:

```bash
docker run -d -p 5678:5678 --name n8n-ui-proxy alpine/socat tcp-listen:5678,fork,reuseaddr tcp-connect:localhost:5678
```

> This makes n8n UI available on `localhost:5678` in Windows browser.

---

## ✅ Post-Install Check

```bash
kubectl get pods
kubectl logs deploy/n8n
kubectl logs deploy/postgres
```

All logs should show ready and connections working. If you see:

```
relation "public.execution_entity" does not exist
```

That’s expected on first startup. n8n will create its DB schema.

---

## 🛑 Minikube Restart (to preserve data)

```bash
minikube stop
minikube start \
  --cpus=6 \
  --memory=12288 \
  --driver=docker \
  --mount \
  --mount-string="/home/ahmetserdargeze/minikube-storage:/mnt/data"
```

No need to reinstall chart; the PVCs will be reused.

---

## 🛠️ Notes

* Never mount volumes from NTFS (e.g., `/mnt/c`) — permissions won’t work.
* Prefer `/home/username` inside WSL for storage.
* Use `securityContext` in PostgreSQL deployment:

```yaml
securityContext:
  runAsUser: 999
  runAsGroup: 999
  fsGroup: 999
```

* Add init container for n8n:

```yaml
initContainers:
  - name: wait-for-postgres
    image: busybox
    command: ['sh', '-c', 'until nc -z postgres 5432; do echo waiting for postgres; sleep 2; done;']
```
