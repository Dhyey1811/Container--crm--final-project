# Django CRM — Compose → Helm (Final Project Guide)

This README walks you from **working Docker Compose** to **Kubernetes + Helm** with
- **Part 1:** Gitea repository & collaborators  
- **Part 2:** **TrueNAS NFS** storage for MySQL  
- **Part 3:** CI that builds & pushes your Docker image (Gitea Registry or Docker Hub)  
- **Part 4:** **Cloudflare Tunnel** (cloudflared) to expose your CRM on the internet  
- **Part 5:** Use **Kompose** to make a Helm chart & deploy to k8s, exposed via Cloudflare

It also includes a **screenshot checklist** that maps to the rubric (20 marks).

> Use this file as your submission guide. Replace placeholders like `<YOUR_DOCKERHUB_USERNAME>` / `<YOUR_DOMAIN>` with your values.

---

## Quick Environment Notes (based on your setup)

- TrueNAS IP: **`10.172.27.21`**
- Pool name: **`pool`**
- Dataset for MySQL data (NFS): **`/mnt/pool/crm-mysql`**
- Bridge interface: **`br0`** (NAS IP moved to the bridge)
- CRM service listens on **port 8080**
- If you can’t run VMs on TrueNAS (`/dev/kvm` missing), use a **VirtualBox VM on your laptop** (Network = **Bridged**) as the NFS client and k8s node.

---

## Part 0 — Run locally with Docker Compose (sanity check)

1) **Patch Django settings** with your hostname (e.g. Codespaces URL or your domain):
   ```diff
   # Add your hosts to the list.
   -ALLOWED_HOSTS = ['localhost', '127.0.0.1']
   -
   +ALLOWED_HOSTS = ['localhost', '127.0.0.1', 'codespace-dev.k3p.dev']
   +CSRF_TRUSTED_ORIGINS = ['https://codespace-dev.k3p.dev']
   ```

   Apply the patch the same way your repo instructions show (copy `webcrm/settings.py` up one level and run `patch < patch-settings.diff`).

2) **Bring up**:
   ```bash
   docker compose up -d --build
   # first run initializes the DB
   docker compose logs -f crm
   ```

3) **Seed initial data** (prints a superuser + password):
   ```bash
   docker compose exec crm sh -lc 'python manage.py setupdata'
   ```

4) Open:
   - Admin: `http://localhost:8080/en/456-admin/`
   - App:   `http://localhost:8080/en/123`

> If you see `mysqlclient` / `libmariadb.so.3` errors, rebuild the image (the Dockerfile must install `mariadb-dev` and `mysqlclient`).

---

## Part 1 — Gitea repo & collaborators (4 marks)

- Create a **new repo** (or clone/“mirror” from the template) on your **Gitea**.  
- **Invite teammates** as **Collaborators** (Repo → Settings → Collaborators).  
- Push your code to that remote and confirm everyone has access.



## Part 2 — TrueNAS NFS for MySQL (4 marks)

### A) TrueNAS (server)

1) **Dataset**
   - Storage → **Datasets** → Pool: `pool` → **Add Dataset** → `crm-mysql`
   - Permissions (lab-friendly): **Open** ACL (Owner: `root`, Group: `wheel`)

2) **NFS Share**
   - Shares → **UNIX (NFS) → Add**
     - **Path:** `/mnt/pool/crm-mysql`
     - **Networks:** add `10.172.27.0/24` (press **Add**)
     - *(If required)* Hosts: add your client VM IP and press **Add**
     - **Advanced Options:** Maproot **User**=`root`, **Group**=`wheel`
   - Services → **NFS** → **Start** and **Enable**

### B) Client test (Linux VM/PC on same LAN)

Use a **VirtualBox** VM on your laptop (Network = **Bridged**) if you can’t make a VM in TrueNAS.

```bash
sudo apt update
sudo apt install -y nfs-common netcat-openbsd

TRUENAS_IP=10.172.27.21
POOL=pool

# NFSv4 port open?
nc -vz $TRUENAS_IP 2049

# Mount + write test
sudo mkdir -p /mnt/nfs-test
sudo mount -t nfs -o vers=4 $TRUENAS_IP:/mnt/$POOL/crm-mysql /mnt/nfs-test
echo "hello from client" | sudo tee /mnt/nfs-test/hello.txt
ls -l /mnt/nfs-test
sudo umount /mnt/nfs-test
```

### C) (Kubernetes) Create PV/PVC for MySQL

`k8s-nfs-mysql.yaml`
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: crm-mysql-pv
spec:
  capacity:
    storage: 10Gi
  accessModes: [ "ReadWriteMany" ]
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 10.172.27.21
    path: /mnt/pool/crm-mysql
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: crm-mysql-pvc
spec:
  accessModes: [ "ReadWriteMany" ]
  resources:
    requests:
      storage: 10Gi
  volumeName: crm-mysql-pv
```

```bash
kubectl apply -f k8s-nfs-mysql.yaml
kubectl get pv,pvc
```


## Part 3 — CI build & push image (4 marks)

You have two choices. Use **A** if your school’s Gitea has **Container Registry** enabled. If not, use **B** (Docker Hub).

### A) Gitea Container Registry (preferred if available)

1) **Personal Access Token** (Settings → Applications → Generate Token)  
   Scopes: **write:package**, **read:package**

2) **Repo Secrets** (Settings → Secrets)
   - `CI_REGISTRY` = e.g. `gitea.school.edu` *or* `gitea.school.edu:3000`
   - `CI_REGISTRY_USER` = your Gitea username
   - `CI_REGISTRY_PASSWORD` = your PAT

3) **Workflow** `.gitea/workflows/build.yml`
```yaml
name: Build & Push Docker Image
on:
  push:
    branches: [ "main" ]
jobs:
  build:
    runs-on: docker
    steps:
      - uses: actions/checkout@v4
      - name: Docker login (Gitea)
        run: |
          echo "${{ secrets.CI_REGISTRY_PASSWORD }}" \
            | docker login "${{ secrets.CI_REGISTRY }}" -u "${{ secrets.CI_REGISTRY_USER }}" --password-stdin
      - name: Build & push
        run: |
          IMAGE="${{ secrets.CI_REGISTRY }}/${{ gitea.repository }}/crm"
          TAG="${{ gitea.sha }}"
          docker build -t "$IMAGE:$TAG" -t "$IMAGE:latest" .
          docker push "$IMAGE:$TAG"
          docker push "$IMAGE:latest"
```

### B) Docker Hub (works everywhere)

1) Create **Access Token** on Docker Hub.  
2) **Repo Secrets:**
   - `DOCKERHUB_USERNAME`  
   - `DOCKERHUB_TOKEN`
3) **Workflow** `.gitea/workflows/build.yml`
```yaml
name: Build & Push to Docker Hub
on:
  push:
    branches: [ "main" ]
jobs:
  build:
    runs-on: docker
    steps:
      - uses: actions/checkout@v4
      - name: Login to Docker Hub
        run: |
          echo "${{ secrets.DOCKERHUB_TOKEN }}" \
            | docker login -u "${{ secrets.DOCKERHUB_USERNAME }}" --password-stdin
      - name: Build & push
        run: |
          IMAGE="docker.io/${{ secrets.DOCKERHUB_USERNAME }}/crm"
          TAG="${{ gitea.sha }}"
          docker build -t "$IMAGE:$TAG" -t "$IMAGE:latest" .
          docker push "$IMAGE:$TAG"
          docker push "$IMAGE:latest"
```



## Part 4 — Cloudflared ingress (4 marks)

Two ways to create the tunnel: **CLI login** or **Dashboard token**. Token is easiest for k8s/compose.

### Option 1: CLI login (creates `~/.cloudflared/cert.pem`)

```bash
cloudflared tunnel login
cloudflared tunnel create crm-tunnel
cloudflared tunnel token crm-tunnel
# copy the long eyJ... token
```

Create a **Public Hostname** in Cloudflare Zero Trust:  
Hostname: `crm.<YOUR_DOMAIN>` → Service:
- k8s: `http://crm.crm.svc.cluster.local:8080`
- compose: `http://crm:8080`

### Option 2: Dashboard-only (no cert needed)

- Zero Trust → **Tunnels** → **Create tunnel** → *crm-tunnel* → copy the **token**.  
- Add **Public Hostname** as above.

#### k8s Deployment (using the token)

```bash
kubectl create namespace crm --dry-run=client -o yaml | kubectl apply -f -
kubectl -n crm create secret generic cloudflared-token \
  --from-literal=TUNNEL_TOKEN='eyJ...your-token...'
```

`k8s/cloudflared-deploy.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: crm
spec:
  replicas: 1
  selector: { matchLabels: { app: cloudflared } }
  template:
    metadata: { labels: { app: cloudflared } }
    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:2024.8.0
          args: ["tunnel","--no-autoupdate","run","--token","$(TUNNEL_TOKEN)"]
          env:
            - name: TUNNEL_TOKEN
              valueFrom:
                secretKeyRef:
                  name: cloudflared-token
                  key: TUNNEL_TOKEN
```

```bash
kubectl apply -f k8s/cloudflared-deploy.yaml
kubectl -n crm get pods
```

#### Docker Compose variant

```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared:2024.8.0
    command: ["tunnel","--no-autoupdate","run","--token","${CLOUDFLARED_TOKEN}"]
    restart: unless-stopped
```

```bash
export CLOUDFLARED_TOKEN='eyJ...'
docker compose up -d cloudflared
```



## Part 5 — Kompose → Helm chart & deploy (4 marks)

### 5.1 Point compose to your built image

In `docker-compose.yml`:
```yaml
services:
  crm:
    image: docker.io/<YOUR_DOCKERHUB_USERNAME>/crm:latest   # or your Gitea registry path
    ports: ["8080:8080"]
    # keep env / depends_on as in your compose
```

### 5.2 Generate a Helm chart

```bash
# install kompose if needed:
# curl -L https://github.com/kubernetes/kompose/releases/latest/download/kompose-linux-amd64 -o kompose && chmod +x kompose && sudo mv kompose /usr/local/bin/

kompose convert -c --chart crm-chart
```

Edit `crm-chart/values.yaml`:
```yaml
image:
  repository: docker.io/<YOUR_DOCKERHUB_USERNAME>/crm
  tag: "latest"

service:
  port: 8080
```

If Kompose generated a MySQL template, ensure it mounts the NFS PVC:
```yaml
volumeMounts:
  - name: mysql-data
    mountPath: /var/lib/mysql
volumes:
  - name: mysql-data
    persistentVolumeClaim:
      claimName: crm-mysql-pvc
```

### 5.3 Apply PV/PVC (from Part 2) & install the chart

```bash
kubectl apply -f k8s-nfs-mysql.yaml
kubectl -n crm get pv,pvc

helm upgrade --install crm ./crm-chart -n crm
kubectl -n crm get pods,svc
```

### 5.4 Cloudflared already points to the Service

- In Cloudflare, Public Hostname Service should be `http://crm.crm.svc.cluster.local:8080`
- Open `https://crm.<YOUR_DOMAIN>` → confirm:
  - `/en/456-admin/` (admin) and `/en/123` (app) both load.

**Optional automation** — `up.yml` / `down.yml`:

`up.yml`
```yaml
- hosts: localhost
  gather_facts: false
  tasks:
    - name: Ensure PV/PVC
      ansible.builtin.command: kubectl apply -f k8s-nfs-mysql.yaml

    - name: Cloudflared secret (uses env)
      ansible.builtin.command: >
        kubectl -n crm create secret generic cloudflared-token
        --from-literal=TUNNEL_TOKEN="${CLOUDFLARED_TOKEN}"
      environment:
        CLOUDFLARED_TOKEN: "{{ lookup('env','CLOUDFLARED_TOKEN') }}"
      failed_when: false

    - name: Cloudflared deploy
      ansible.builtin.command: kubectl apply -f k8s/cloudflared-deploy.yaml

    - name: Install chart
      ansible.builtin.command: helm upgrade --install crm ./crm-chart -n crm
```

`down.yml`
```yaml
- hosts: localhost
  gather_facts: false
  tasks:
    - name: Uninstall chart
      ansible.builtin.command: helm uninstall crm -n crm
      failed_when: false
    - name: Remove cloudflared
      ansible.builtin.command: kubectl -n crm delete deploy cloudflared secret cloudflared-token
      failed_when: false
    - name: Remove PVC only (keep PV data)
      ansible.builtin.command: kubectl -n crm delete pvc crm-mysql-pvc
      failed_when: false
```

Run:
```bash
ansible-playbook up.yml
# ... later ...
ansible-playbook down.yml
```

---



Good luck!
