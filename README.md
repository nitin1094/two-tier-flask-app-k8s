# Two-Tier Flask + MySQL App on Kubernetes

Deploying a **two-tier web application** — a **Python / Flask** app backed by a **MySQL** database — on Kubernetes, with manifests for **both** a self-managed **kubeadm** cluster (NodePort + hostPath storage) and **Amazon EKS** (cloud LoadBalancer + Secret/ConfigMap). Also containerized with Docker and wired into a Jenkins CI pipeline.

The Flask app stores and displays user-submitted messages; it finds the database through the `mysql` Service by name (service discovery), so it keeps working as pods are recreated.

![Kubernetes](https://img.shields.io/badge/Kubernetes-Deploy-326CE5?logo=kubernetes&logoColor=white)
![kubeadm](https://img.shields.io/badge/Cluster-kubeadm-326CE5)
![AWS EKS](https://img.shields.io/badge/AWS-EKS-FF9900?logo=amazoneks&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose%20%2B%20Image-2496ED?logo=docker&logoColor=white)
![Flask](https://img.shields.io/badge/App-Flask-000000?logo=flask&logoColor=white)
![MySQL](https://img.shields.io/badge/Database-MySQL-4479A1?logo=mysql&logoColor=white)
![Jenkins](https://img.shields.io/badge/CI-Jenkins-D24939?logo=jenkins&logoColor=white)

---

## Architecture

```mermaid
flowchart TD
    U["🌐 Browser"] --> SVC["two-tier-app-service<br/>NodePort (kubeadm) / LoadBalancer (EKS)"]

    subgraph K8S["Kubernetes cluster"]
        SVC --> APP["two-tier-app<br/>Flask :5000"]
        APP -->|"host: mysql"| DB["mysql Service · ClusterIP :3306"]
        DB --> POD["MySQL pod"]
        POD -->|"kubeadm: PVC → hostPath PV"| PV[("/var/lib/mysql")]
        SEC["Secret / env<br/>MySQL credentials"] -.-> POD
    end
```

| Tier | Component | kubeadm (`k8s/`) | EKS (`eks-manifests/`) |
|------|-----------|------------------|------------------------|
| Application | Flask app (`:5000`) | Service type **NodePort** (`30004`) | Service type **LoadBalancer** |
| Data | MySQL (`:3306`) | ClusterIP + hostPath **PV/PVC** | ClusterIP + **Secret** + init **ConfigMap** |

---

## Repository layout

```
.
├── app.py                     # Flask app (reads MySQL config from env vars)
├── templates/index.html       # the frontend page
├── message.sql                # creates the `messages` table
├── requirements.txt           # Python dependencies
├── Dockerfile                 # container build (+ Dockerfile-multistage variant)
├── docker-compose.yml         # local two-container run (app + MySQL)
├── Makefile                   # build / run / clean shortcuts around compose
├── Jenkinsfile                # CI: clone → Trivy scan → build → push → deploy
├── k8s/                       # ← kubeadm deployment (NodePort + hostPath PV)
│   ├── mysql-pv.yml / mysql-pvc.yml
│   ├── mysql-deployment.yml / mysql-svc.yml
│   ├── two-tier-app-deployment.yml / two-tier-app-svc.yml
│   └── two-tier-app-pod.yml   # (alternative: run the app as a bare Pod)
└── eks-manifests/             # ← Amazon EKS deployment (LoadBalancer + Secret)
    ├── mysql-secrets.yml / mysql-configmap.yml
    ├── mysql-deployment.yml / mysql-svc.yml
    └── two-tier-app-deployment.yml / two-tier-app-svc.yml
```

---

## Prerequisites

Before you begin, make sure you have the following installed:

- **Docker** (and the **Docker Compose** plugin — `docker compose version`)
- **Git** (optional, for cloning the repository)
- For the Kubernetes part: a running cluster and **`kubectl`** — either a self-managed
  cluster (e.g. built with `kubeadm`) or **Amazon EKS** (with the AWS CLI / `eksctl`)

---

# Part 1 — Flask App with MySQL Docker Setup

Run the whole two-tier app locally in containers, either with Docker Compose or with plain `docker run` commands.

## 1. Clone the repository

```bash
git clone https://github.com/nitin1094/two-tier-flask-app-k8s.git
cd two-tier-flask-app-k8s
```

## 2. Create a `.env` file

The Compose file runs MySQL as `user: "${UID}:${GID}"` so the bind-mounted data
directory isn't owned by root. Docker Compose reads these from a `.env` file in the
project directory, so create one:

```bash
touch .env
```

Add your user/group IDs to it:

```bash
# Linux / macOS — fill in the real values:
#   id -u   ->  UID
#   id -g   ->  GID
UID=1000
GID=1000
```

> On **Linux/macOS** you can generate it in one line:
> ```bash
> printf "UID=%s\nGID=%s\n" "$(id -u)" "$(id -g)" > .env
> ```
> On **Windows** (Docker Desktop) these IDs aren't meaningful — leave `UID=1000` /
> `GID=1000`. If MySQL then fails to start on the bind mount, delete the
> `user: "${UID}:${GID}"` line from `docker-compose.yml`.

## 3. Build the application image

The Compose file references the image **by name only** (it has no `build:` section),
so `docker compose up --build` will *not* build anything — build and tag it yourself first:

```bash
docker build -t nitin1094/two-tier-flask-app:latest .
```

<details>
<summary>Two image tags are used in this repo</summary>

| Consumer | Expected image |
|---|---|
| `docker-compose.yml` | `nitin1094/two-tier-flask-app:latest` |
| Kubernetes manifests (`k8s/`, `eks-manifests/`) | `nitin1094/flaskapp:latest` |

Tag one build for both so either path works:
```bash
docker build -t nitin1094/two-tier-flask-app:latest .
docker tag nitin1094/two-tier-flask-app:latest nitin1094/flaskapp:latest
```

There is also a smaller multi-stage build:
```bash
docker build -f Dockerfile-multistage -t nitin1094/two-tier-flask-app:slim .
```
</details>

## 4. Option A — Run with Docker Compose (recommended)

```bash
docker compose up -d          # start MySQL + the Flask app in the background
docker compose ps             # both containers should be "Up"
```

Compose creates a `twotier` network, starts `mysql:5.7` (database **`devops`**, root
password **`root`**), mounts `./mysql-data` for persistence, auto-loads `message.sql`
to create the `messages` table, then starts the Flask app on port **5000**.

**Access the app:** <http://localhost:5000>

Open it in your browser, type a message into the form, and submit — it's written to
MySQL and appears in the list on reload.

**Follow the logs:**

```bash
docker compose logs -f flask-app
docker compose logs -f mysql
```

**Stop / clean up:**

```bash
docker compose down           # stop and remove the containers + network
docker compose down -v        # ...and also remove named volumes
```

> The `./mysql-data` folder is a bind mount on your host, so your data survives
> `docker compose down`. Delete that folder to start from an empty database.

## 5. Option B — Run without Docker Compose

Useful for understanding what Compose does for you. Both containers must share a
network so they can reach each other by name.

**i) Create the network**

```bash
docker network create twotier
```

**ii) Start the MySQL container**

```bash
docker run -d \
    --name mysql \
    --network=twotier \
    -v mysql-data:/var/lib/mysql \
    -e MYSQL_DATABASE=mydb \
    -e MYSQL_ROOT_PASSWORD=admin \
    -p 3306:3306 \
    mysql:5.7
```

**iii) Start the Flask container**

```bash
docker run -d \
    --name flaskapp \
    --network=twotier \
    -e MYSQL_HOST=mysql \
    -e MYSQL_USER=root \
    -e MYSQL_PASSWORD=admin \
    -e MYSQL_DB=mydb \
    -p 5000:5000 \
    nitin1094/flaskapp:latest
```

`MYSQL_HOST=mysql` is the **container name** — Docker's embedded DNS resolves it on
the `twotier` network, which is the same idea as a Kubernetes Service name later on.

**iv) Verify**

```bash
docker ps                     # both containers running?
docker logs flaskapp          # look for "Connected to database."
```

Then open <http://localhost:5000>.

**v) Clean up**

```bash
docker rm -f flaskapp mysql
docker network rm twotier
docker volume rm mysql-data   # optional: delete the database data
```

## The `messages` table

The app stores messages in a single table:

```sql
CREATE TABLE messages (
    id INT AUTO_INCREMENT PRIMARY KEY,
    message TEXT
);
```

You normally **don't** need to create it by hand — it's created for you in two ways:

- **Compose:** `message.sql` is mounted into `/docker-entrypoint-initdb.d/`, so MySQL
  runs it the first time the database is initialised.
- **The app:** `init_db()` in `app.py` runs `CREATE TABLE IF NOT EXISTS` on startup.

If you ever need to create it manually (e.g. the plain-Docker path with a pre-existing
volume):

```bash
docker exec -it mysql mysql -uroot -padmin mydb \
  -e "CREATE TABLE IF NOT EXISTS messages (id INT AUTO_INCREMENT PRIMARY KEY, message TEXT);"
```

## Environment variables

`app.py` reads all database settings from the environment (defaults in brackets):

| Variable | Purpose | Default |
|---|---|---|
| `MYSQL_HOST` | Database hostname — a container name, Docker Compose service, or K8s Service | `localhost` |
| `MYSQL_USER` | MySQL user | `default_user` |
| `MYSQL_PASSWORD` | MySQL password | `default_password` |
| `MYSQL_DB` | Database name | `default_db` |

## Routes

| Method | Path | Description |
|---|---|---|
| `GET` | `/` | Renders `index.html` with all stored messages |
| `POST` | `/submit` | Saves a new message (form field `new_message`), returns JSON |

## Push the image to a registry

Needed before deploying to Kubernetes, since the cluster pulls the image:

```bash
docker login
docker push nitin1094/two-tier-flask-app:latest
docker push nitin1094/flaskapp:latest
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| `docker compose up --build` doesn't rebuild the app | Expected — the service has no `build:` section. Run `docker build -t nitin1094/two-tier-flask-app:latest .` first (step 3). |
| Compose warns `The "UID" variable is not set` | Create the `.env` file from step 2. |
| MySQL container exits / permission denied on `./mysql-data` | The `user: "${UID}:${GID}"` mapping doesn't match the folder owner. Fix the IDs in `.env`, `rm -rf ./mysql-data`, or remove the `user:` line. |
| `flask-app` shows **(unhealthy)** but the site works | The Compose health-check curls `/health`, which this app doesn't implement. Harmless — ignore it, or add a `/health` route to `app.py`. |
| App logs `Could not connect to database.` | MySQL wasn't ready yet (it can take ~30–60s on first run) or the credentials don't match. Check `docker compose logs mysql` and that `MYSQL_USER`/`MYSQL_PASSWORD`/`MYSQL_DB` line up. |
| Port already in use | Something else is on 5000/3306 — change the left-hand side of the port mapping. |

---

# Part 2 — Deploy on Kubernetes

> **Before you start:** build and push the app image (steps 3 and *Push the image* above),
> and make sure the `image:` field in the `two-tier-app` manifests points at it.

## On a kubeadm cluster

```bash
cd k8s
kubectl apply -f mysql-pv.yml -f mysql-pvc.yml                          # storage
kubectl apply -f mysql-deployment.yml -f mysql-svc.yml                  # database
kubectl apply -f two-tier-app-deployment.yml -f two-tier-app-svc.yml    # app
kubectl get pods,svc
# open http://<worker-node-ip>:30004/
```

See [`k8s/README.md`](k8s/README.md) for the detailed walkthrough.

## On Amazon EKS

```bash
cd eks-manifests
kubectl apply -f mysql-configmap.yml -f mysql-secrets.yml
kubectl apply -f mysql-deployment.yml -f mysql-svc.yml
kubectl apply -f two-tier-app-deployment.yml -f two-tier-app-svc.yml
kubectl get svc two-tier-app-service   # wait for the LoadBalancer EXTERNAL-IP, then open it
```

---

## What this project demonstrates

- **Containerizing** a Python/Flask + MySQL app (single-stage and multi-stage Dockerfiles)
- Running a multi-container app with **Docker Compose** and with plain Docker networking
- **Two-tier deployment** on Kubernetes: Deployments, Services, and cross-tier **service discovery** by Service name
- Two exposure models: **NodePort** on a self-managed cluster vs a cloud **LoadBalancer** on EKS
- **Persistent storage** (PV/PVC, hostPath) so the database survives pod restarts
- Managing configuration with **Secrets** and **ConfigMaps** (EKS variant)
- **CI** with Jenkins: source scan (Trivy) → build → push to a registry → deploy

---

## Security note

The manifests and Compose file ship with **demo database credentials** so the project
runs out of the box. They are throwaway values — replace them with your own before any
real use. Kubernetes Secrets are only base64-encoded (not encrypted), so never commit
real credentials; use a sealed-secrets tool or an external secrets manager in production.

Also note the app interpolates form input via parameterised queries — keep it that way,
and validate/sanitise any new inputs to avoid SQL injection.
