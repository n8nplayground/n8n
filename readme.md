# n8n + PostgreSQL Helm Kurulumu (Windows 11 + WSL2 + Minikube + Helm)

Bu rehber, Windows 11 üzerinde WSL2 kullanarak Minikube ortamında `n8n` ve `PostgreSQL` servislerinin Helm chart ile **persistent volume (PV)** desteğiyle nasıl kurulacağını ve karşılaşılan problemlerin nasıl çözüleceğini adım adım açıklar.

---

## 💡 Gereksinimler

* Windows 11
* WSL2 (Ubuntu tavsiye edilir)
* Docker Desktop (WSL entegrasyonu açık)
* Helm
* Minikube
* `kubectl`

---

## 📦 Adım 1: Minikube Ortamı Hazırlığı

### WSL klasör yapısı oluştur

```bash
mkdir -p ~/minikube-storage/n8n
mkdir -p ~/minikube-storage/postgres
```

> Bu klasörler persistent volume olarak mount edilecektir. Windows sürücüsü yerine **WSL dosya sistemi kullanılmalıdır** (NTFS izin hatası verir).

### İzinleri düzelt

```bash
sudo chown -R 999:999 ~/minikube-storage/postgres
sudo chmod -R 700 ~/minikube-storage/postgres

sudo mkdir -p ~/minikube-storage/n8n
sudo chown -R 1000:1000 ~/minikube-storage/n8n
```

---

## 🚀 Adım 2: Minikube Başlatma

```bash
minikube start --cpus=6 --memory=12288 --mount --mount-string="$HOME/minikube-storage:/mnt/data"
```

> `--mount-string` ile WSL içindeki volume'ler Minikube'a `/mnt/data` olarak aktarılır.

---

## 🛠️ Adım 3: Helm Chart Yapılandırması

### `values.yaml` örneği

```yaml
n8n:
  image: n8nio/n8n
  tag: latest
  port: 5678
  mountPath: /home/node/.n8n
  volumePath: /mnt/data/n8n

postgres:
  image: postgres
  tag: 17
  port: 5432
  database: n8ndb
  host: postgres
  username: n8n
  password: n8npass
  mountPath: /var/lib/postgresql/data
  volumePath: /mnt/data/postgres
```

---

## 📁 Adım 4: PersistentVolume ve PersistentVolumeClaim

### `postgres-pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data/postgres
  persistentVolumeReclaimPolicy: Retain
```

### `postgres-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

---

## 🧱 Adım 5: PostgreSQL Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      securityContext:
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
      containers:
        - name: postgres
          image: {{ .Values.postgres.image }}:{{ .Values.postgres.tag }}
          ports:
            - containerPort: {{ .Values.postgres.port }}
          env:
            - name: POSTGRES_USER
              value: {{ .Values.postgres.username }}
            - name: POSTGRES_PASSWORD
              value: {{ .Values.postgres.password }}
            - name: POSTGRES_DB
              value: {{ .Values.postgres.database }}
          volumeMounts:
            - name: postgres-storage
              mountPath: {{ .Values.postgres.mountPath }}
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc
```

---

## 🧹 Adım 6: n8n Deployment

### Init Container ile Postgres hazır bekletmesi

```yaml
initContainers:
  - name: wait-for-postgres
    image: busybox
    command: ['sh', '-c', 'until nc -z postgres 5432; do echo waiting for postgres; sleep 2; done;']
```

---

## 🔁 Adım 7: Uygulamaları Yükleme

```bash
kubectl apply -f postgres-pv.yaml
kubectl apply -f postgres-pvc.yaml
helm upgrade --install n8n ./n8n-helm -f values.yaml
```

---

## 🛠️ Karşılaşılan Hatalar ve Çözümleri

### ❌ `chmod: changing permissions of '/var/lib/postgresql/data': Operation not permitted`

**Sebep:** NTFS veya root olmayan klasör izin problemi.

**Çözüm:**

```bash
sudo chown -R 999:999 ~/minikube-storage/postgres
chmod -R 700 ~/minikube-storage/postgres
```

---

### ❌ `role "n8n" does not exist`

**Sebep:** PVC klasörü silinmeden deployment tekrarlandı → `initdb` çalışmadı.

**Çözüm:**

```bash
kubectl delete pod -l app=postgres
kubectl delete pvc postgres-pvc
kubectl delete pv postgres-pv
sudo rm -rf ~/minikube-storage/postgres/*
```

---

### ❌ n8n `Completed` oluyor

**Sebep:** Postgres'e bağlanamadan çıkıyor.

**Çözüm:**

* Init container (`wait-for-postgres`) ekle.
* DB bilgileri `values.yaml` ile tam uyumlu olmalı.

---

## 🌐 n8n Arayüzünüze Erişim

```bash
minikube service n8n --url
```

> Çıkan URL’yi tarayıcıda açabilirsiniz: `http://<minikube-ip>:5678`

---

## ♻️ Minikube Yeniden Başlatma

```bash
minikube stop
minikube start --mount --mount-string="$HOME/minikube-storage:/mnt/data"
```

Helm chart güncellemesi için:

```bash
helm upgrade --install n8n ./n8n-helm -f values.yaml
```

---

## ✅ Ekstra Notlar

* Volume mount path her zaman WSL içinde kalmalı.
* PostgreSQL podu `CrashLoopBackOff` olursa loglara bak: `kubectl logs <postgres-pod>`

---

## 💜 Temizlik

```bash
kubectl delete all --all
kubectl delete pvc --all
kubectl delete pv --all
sudo rm -rf ~/minikube-storage/*
```

---

```
}

```
