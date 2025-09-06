# Spring Boot + PostgreSQL on Kubernetes (k3s) — Local Image (no registry)

This bundle deploys:
- **PostgreSQL** as a **StatefulSet** with **headless Service**
- **Spring Boot** app as a **Deployment**
- **ClusterIP Service** + **Ingress** (Traefik on k3s)
- **Static NFS PV** bound to the Postgres PVC

> This version is tailored for **Path A** you followed:
> Build Docker image on the k3s node and import it into k3s’ containerd.

---

## 0) Prereqs

**On NFS server (example 10.0.0.10)**  
Export: `/srv/nfs/data/pgdata` (rw).

**On k3s node (example 10.0.0.20)**
- k3s installed and `kubectl` working
- Install NFS client utilities:
  ```bash
  sudo apt update && sudo apt install -y nfs-common
  ```
- Build & import your Spring image locally:
  ```bash
  # From your Spring repo on the k3s node
  docker build -t spring-sample:local -f Dockerfile .
  docker save spring-sample:local | sudo k3s ctr images import -
  sudo k3s ctr images ls | grep spring-sample
  ```

> The Deployment here expects the image **spring-sample:local** and uses `imagePullPolicy: IfNotPresent`.

---

## 1) Minimal edits before applying

- **NFS server IP/path**: edit `k8s/storage/pv-nfs.yaml`
  ```yaml
  nfs:
    server: 10.0.0.10
    path: /srv/nfs/data/pgdata
  ```

- If you used a **different local image tag**, update `k8s/app/spring-deployment.yaml`:
  ```yaml
  image: spring-sample:local
  imagePullPolicy: IfNotPresent
  ```

---

## 2) Deploy in order

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/security/secret-db.yaml

kubectl apply -f k8s/storage/pv-nfs.yaml
kubectl apply -f k8s/db/postgres-service-headless.yaml
kubectl apply -f k8s/db/postgres-statefulset.yaml

# wait for postgres Ready
kubectl -n spring-demo rollout status statefulset/postgres

kubectl apply -f k8s/app/spring-service.yaml
kubectl apply -f k8s/app/spring-deployment.yaml
kubectl apply -f k8s/ingress.yaml
```

---

## 3) Access

Add host mapping on your laptop (or the k3s node if you browse from there):
```
echo "<K3S_NODE_IP> spring.local" | sudo tee -a /etc/hosts
```
Open: `http://spring.local/swagger-ui/index.html` (or your app root).

---

## 4) Verify persistence & wiring

```bash
kubectl -n spring-demo get pv,pvc
kubectl -n spring-demo get pods
kubectl -n spring-demo logs deploy/spring-app --tail=100
kubectl -n spring-demo exec sts/postgres -- psql -U appuser -d demo -c '\l'
```
On the **NFS server**, check files under `/srv/nfs/data/pgdata`.

---

## 5) Cleanup
```bash
kubectl delete -f k8s --recursive
```

---

## Diagram
Ingress (Traefik) → Spring Service → Spring Deployment (Pod)
Headless Service → PostgreSQL StatefulSet (Pod) → PVC → PV → NFS
