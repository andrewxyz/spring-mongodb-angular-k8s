# Deploying spring boot-mongodb-angular apps to Kubernetes (kind)

Browser → `http://mylocal.localhost` → Traefik (Ingress) → `/api` → backend ×2 → MongoDB (StatefulSet + disk)
                                                    → `/`    → frontend ×2 (nginx)

Namespace: `demo-namespace`. Cluster: `kind-demo-cluster`.

## Architecture

A three-tier web app (Angular UI → Spring Boot REST API → MongoDB) running in a local kind cluster, with Traefik as the single entry point.

### Overview

```
                        Your PC
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │  Browser ── http://mylocal.localhost ──► port 80 (forwarded by kind)        │
 │                                                                             │
 │  kind cluster "demo-cluster" (each node is a Docker container)              │
 │ ┌─────────────────────────────────────────────────────────────────────────┐ │
 │ │ control-plane node                                                      │ │
 │ │   ┌──────────────────────────────────────────┐                          │ │
 │ │   │ namespace: traefik                       │                          │ │
 │ │   │ Traefik (Ingress controller)  hostPort 80│                          │ │
 │ │   └───────────────┬──────────────────────────┘                          │ │
 │ └───────────────────┼─────────────────────────────────────────────────────┘ │
 │                     │  rules from Ingress "demo-ingress"                    │
 │ ┌───────────────────┼─────────────────────────────────────────────────────┐ │
 │ │ worker nodes      │          namespace: demo-namespace                  │ │
 │ │        /api ──────┴────── /                                             │ │
 │ │          │                 │                                            │ │
 │ │          ▼                 ▼                                            │ │
 │ │  backend-service     frontend-service        (Services, ClusterIP)      │ │
 │ │     :8080                :80                                            │ │
 │ │     │    │               │    │                                         │ │
 │ │  ┌──┴─┐┌─┴──┐         ┌──┴─┐┌─┴──┐                                      │ │
 │ │  │ be ││ be │         │ fe ││ fe │     Deployments, replicas = 2        │ │
 │ │  │pod ││pod │         │pod ││pod │     be = Spring Boot, fe = nginx     │ │
 │ │  └──┬─┘└─┬──┘         └────┘└────┘                                      │ │
 │ │     └──┬─┘                                                              │ │
 │ │        ▼  mongodb:27017 (headless Service)                              │ │
 │ │   ┌──────────┐      ┌────────────────────┐                              │ │
 │ │   │mongodb-0 │─────►│ PVC data-mongodb-0 │   StatefulSet, replicas = 1  │ │
 │ │   └──────────┘      │ 1Gi (local-path)   │                              │ │
 │ │                     └────────────────────┘                              │ │
 │ │   mongo-client (plain Pod, for debugging)                               │ │
 │ └─────────────────────────────────────────────────────────────────────────┘ │
 └─────────────────────────────────────────────────────────────────────────────┘
```

### Components

| Layer | Component | Kubernetes object | Replicas | Technology |
|---|---|---|---|---|
| Edge | Traefik | Deployment (Helm chart), IngressClass `traefik` | 1 | Traefik v3, `hostPort` 80/443 on the control-plane node |
| Routing | `demo-ingress` | Ingress | n/a | host `mylocal.localhost`; `/api` → backend, `/` → frontend |
| Presentation | `frontend` | Deployment + Service `frontend-service` (:80) | 2 | Angular 17 static files served by nginx |
| Application | `backend` | Deployment + Service `backend-service` (:8080) | 2 | Spring Boot 3.5 / Java 25, REST API under `/api/tutorials` |
| Data | `mongodb` | StatefulSet + headless Service `mongodb` (:27017) | 1 | MongoDB 7, database `belajar_db` |
| Storage | `data-mongodb-0` | PersistentVolumeClaim (1Gi, StorageClass `standard`) | n/a | kind's local-path provisioner |
| Config | `backend-config`, `frontend-nginx-config` | ConfigMaps | n/a | env vars for the backend; `default.conf` file for nginx |
| Secrets | `mongodb-secret` | Secret | n/a | MongoDB root username and password |
| Debug | `mongo-client` | plain Pod | 1 | `mongosh`, started by hand, not self-healing |

### How requests flow

**1. Opening the page** (`GET /`)
```
Browser → Traefik → frontend-service → one of the 2 nginx pods → index.html + JS/CSS
```
Any path with no file on disk (such as `/tutorials/42`) gets `index.html` from nginx's `try_files` fallback, and the Angular router takes over in the browser.

**2. Using the app** (`GET/POST/PUT/DELETE /api/tutorials...`)
```
Angular (in the browser) → Traefik → backend-service → one of the 2 backend pods → mongodb:27017 → disk
```
The Angular code calls the relative URL `/api/tutorials`, so page and API share one origin (`mylocal.localhost`). That means no CORS and no hardcoded backend address.

### How configuration reaches the pods

```
ConfigMap backend-config ──envFrom──────────────► backend pods  SPRING_DATA_MONGODB_HOST / PORT / DATABASE / AUTHENTICATION_DATABASE
Secret   mongodb-secret  ──secretKeyRef─────────► backend pods  SPRING_DATA_MONGODB_USERNAME / PASSWORD
Secret   mongodb-secret  ──envFrom──────────────► mongodb-0     MONGO_INITDB_ROOT_USERNAME / PASSWORD (creates the root user on first start)
ConfigMap frontend-nginx-config ──volume mount──► frontend pods /etc/nginx/conf.d/default.conf
```

Spring Boot turns the environment variables into its `spring.data.mongodb.*` properties. The same code runs locally (defaults: `localhost:27017`, no authentication) and in the cluster, with no code change.

### Health checks

| Pod | Check | Purpose |
|---|---|---|
| backend | `startupProbe`, `readinessProbe`, `livenessProbe` on `/actuator/health/liveness` and `/readiness` | Wait for the JVM to boot; send traffic only to ready pods; restart a stuck pod |
| frontend | `readinessProbe`, `livenessProbe` on `GET /` | Same, for nginx |
| mongodb | `mongosh` ping | Same, for MongoDB |

The actuator endpoints are only reachable from inside the cluster, because the Ingress routes just `/api` to the backend.

### Design decisions

| Decision | Why |
|---|---|
| One host, path-based routing (`/api` vs `/`) | One origin removes CORS and the hardcoded API URL; the app is the same in every environment. |
| Backend and frontend as **Deployments** with 2 replicas | They're stateless, so any replica can answer. Pods are replaced if they die, and updates roll out one pod at a time. |
| MongoDB as a **StatefulSet** with 1 replica and a PVC | Stable pod name and its own disk, so data survives pod restarts. Several plain mongod pods would be separate databases; a real multi-pod setup needs a replica set. |
| Headless Service for MongoDB | A StatefulSet needs stable per-pod DNS (`mongodb-0.mongodb`); the backend just uses the name `mongodb`. |
| Only `/api` and `/` are exposed | MongoDB has no route and is unreachable from outside the cluster; actuator endpoints stay internal. |
| Traefik with `hostPort` instead of a LoadBalancer | kind has no cloud load balancer; `hostPort` plus kind's port mapping gives the same result locally. |
| Standard `Ingress` instead of Traefik's own `IngressRoute` | Works with any Ingress controller, so the manifests are portable. |
| Separate namespace `demo-namespace` | Keeps this project's objects apart; `kubectl delete -k k8s/` removes them cleanly. |
| Fixed image tag + `imagePullPolicy: IfNotPresent` | The images are loaded straight into kind with `kind load`; there's no registry. |

### Build and deploy flow

```
source code ──docker build──► local image ──kind load──► images inside each kind node ──kubectl apply -k──► running pods
(spring-boot-data-mongodb/                                                            (manifests in k8s/)
 angular-crud/)
```

Both Dockerfiles are multi-stage: a build stage (Maven + JDK 25, or Node 20) produces the artifact, and a small runtime stage (JRE 25, or nginx) is what actually runs, with the backend as a non-root user.

### Where each piece is defined

| Piece | File |
|---|---|
| Cluster shape and port mapping | [kind-config.yaml](kind-config.yaml) |
| Traefik settings | [traefik-values.yaml](traefik-values.yaml) |
| Everything applied by `kubectl apply -k` | [kustomization.yaml](kustomization.yaml) |
| Routing rules | [ingress.yaml](ingress.yaml) |
| MongoDB (Secret, Service, StatefulSet) | [mongodb/](mongodb/) |
| Backend (ConfigMap, Deployment, Service) | [backend/](backend/) |
| Frontend (ConfigMap, Deployment, Service) | [frontend/](frontend/) |
| Debug Pod | [debug/mongo-client-pod.yaml](debug/mongo-client-pod.yaml) |

### Limits of this setup

- One Traefik replica and one MongoDB replica are single points of failure; fine for learning, not for production.
- No HTTPS: port 443 is open but there's no certificate.
- MongoDB credentials are plain text in `mongodb/secret.yaml`.
- No NetworkPolicies: any pod in the cluster can reach MongoDB.
- The data lives on a kind node's disk. Deleting the cluster deletes it.

## First-time setup (run from the repo root)

```bash
# 1. Cluster (ports 80/443 on your PC are forwarded to the control-plane node)
kind create cluster --name demo-cluster --config k8s/kind-config.yaml

# 2. Ingress controller (Traefik)
helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik -n traefik --create-namespace -f k8s/traefik-values.yaml

# 3. Build the images
docker build -t belajar-backend:1.0.0  spring-boot-data-mongodb
docker build -t belajar-frontend:1.0.0 angular-crud

# 4. Copy the images into the kind nodes (they can't see your local Docker images)
kind load docker-image belajar-backend:1.0.0 belajar-frontend:1.0.0 --name demo-cluster

# 5. Deploy everything
kubectl apply -k k8s/

# 6. Watch it start
kubectl get pods -n demo-namespace -w
```

Open **http://mylocal.localhost**.

## Everyday commands

```bash
kubectl get all,ing,pvc,cm,secret -n demo-namespace     # what exists
kubectl logs -n demo-namespace deploy/backend -f        # backend logs
kubectl describe pod <name> -n demo-namespace           # why is a pod not starting?
kubectl scale deploy/backend -n demo-namespace --replicas=3

# MongoDB shell via the debug pod
kubectl exec -it mongo-client -n demo-namespace -- sh -c \
  'mongosh "mongodb://$MONGO_INITDB_ROOT_USERNAME:$MONGO_INITDB_ROOT_PASSWORD@mongodb:27017/belajar_db?authSource=admin"'
```

## After changing app code

Use a new tag each time; reusing `1.0.0` with `IfNotPresent` keeps the old image.

```bash
docker build -t belajar-backend:1.0.1 spring-boot-data-mongodb
kind load docker-image belajar-backend:1.0.1 --name demo-cluster
kubectl set image deploy/backend backend=belajar-backend:1.0.1 -n demo-namespace   # rolling update
```
(Also update the `image:` line in `backend/deployment.yaml` so the file matches.)

## Publishing the images to Docker Hub

Step-by-step commands for pushing `belajar-backend` and `belajar-frontend` to Docker Hub, and for making the cluster pull them from there: [dockerhub.md](dockerhub.md).

## Things to try (to see Kubernetes at work)

| Experiment | Expected |
|---|---|
| `kubectl delete pod <a backend pod>` | A new one starts; the app keeps answering (the other replica serves). |
| `kubectl delete pod mongodb-0` | Comes back with the same name and the same disk; your data is still there. |
| `kubectl delete -k k8s/` then `kubectl apply -k k8s/` | Everything is recreated; data is still there because the PVC isn't deleted. |

## Traefik explained (for Kubernetes beginners)

### 1. The problem it solves

Your pods live on a private network inside the cluster. Their IPs (like `10.244.1.3`) can't be reached from your browser, and they change every time a pod is recreated.

A **Service** gives pods a stable name, but only inside the cluster. So how does `http://mylocal.localhost` from your browser reach the right pod? You need something at the edge of the cluster that:

1. listens on a real port (80) that the outside world can reach,
2. looks at each request ("which host? which path?"),
3. forwards it to the right place inside.

That something is a **reverse proxy**. Traefik is a reverse proxy, and it's built to run inside Kubernetes.

### 2. Ingress vs. Ingress controller

Kubernetes splits this job in two, and this is the most confusing part for beginners.

| | What it is | In this project |
|---|---|---|
| **Ingress** | A Kubernetes object that holds routing *rules*, written as YAML. It's only data, and it does nothing by itself | [ingress.yaml](ingress.yaml) |
| **Ingress controller** | A program that *reads* those rules and does the actual routing | **Traefik** |

An analogy: the Ingress is a seating chart, and Traefik is the host who reads it and walks guests to their tables. Without a host, the chart is just paper. That's why, with Traefik installed but no Ingress applied yet, a request to port 80 gets a 404. Traefik is running but has no rules.

Kubernetes ships **no** controller by default. You choose and install one. Common choices are Traefik, NGINX, HAProxy and cloud load balancers. This project uses Traefik because ingress-nginx stopped being maintained in March 2026, and Traefik installs easily with Helm.

### 3. How Traefik knows what to do

Traefik has no config file listing your apps. It talks to the **Kubernetes API** and watches for changes, continuously:

```
You: kubectl apply -k k8s/
        │
        ▼
Kubernetes API stores the Ingress "demo-ingress"
        │   (Traefik is subscribed to changes)
        ▼
Traefik notices → reads the rules → looks up the Service's pods
        │
        ▼
Traefik updates its routing table in memory. No restart, no downtime.
```

Add a new Ingress and Traefik picks it up in about a second. Delete a pod and Traefik notices the endpoint change and stops sending traffic there. This is why it needs permissions: the Helm chart creates a ServiceAccount and RBAC rules so that it can *read* Ingresses, Services and pods.

### 4. Traefik's own vocabulary

Inside, Traefik translates your Ingress into four concepts. You'll see these words in its docs and logs:

| Term | Meaning | This setup |
|---|---|---|
| **EntryPoint** | A port Traefik listens on | `web` = port 80, `websecure` = port 443 (HTTPS) |
| **Router** | A matching rule plus a destination | `Host(mylocal.localhost) && PathPrefix(/api)` |
| **Middleware** | Optional steps between match and destination: strip a prefix, add headers, rate-limit, redirect to HTTPS | None yet |
| **Service** | The group of backends to send traffic to | the backend pods |

The Ingress becomes two routers. When both match, **the more specific (longer) rule wins**, which is why `/api/...` goes to the backend and everything else falls through to the frontend:

```
request: mylocal.localhost/api/tutorials  → matches /api AND /  → /api is longer → backend
request: mylocal.localhost/tutorials/42   → matches only /      → frontend (nginx)
request: other.localhost/                 → matches no host     → 404 from Traefik
```

One detail: Traefik uses the Service only to *discover* which pods exist. It then sends requests straight to the pod IPs and does the load-balancing itself. So with 2 backend replicas it alternates between them.

### 5. How a request reaches Traefik in this cluster

This takes the most setup, because kind runs "nodes" as Docker containers on your PC:

```
Browser: http://mylocal.localhost
   │  *.localhost always resolves to 127.0.0.1 (built into browsers/curl)
   ▼
Your PC, port 80
   │  kind-config.yaml: extraPortMappings 80 → 80
   ▼
The control-plane node (a Docker container)
   │  traefik-values.yaml: hostPort 80 → container port 8000
   ▼
Traefik pod  (pinned to this node with nodeSelector ingress-ready=true)
   │  matches the Ingress rules
   ▼
backend pod or frontend pod
```

Each piece in [traefik-values.yaml](traefik-values.yaml) has a reason:

- `hostPort: 80`: opens port 80 directly on the node. Real clouds would use a `LoadBalancer` Service (a cloud-provided public IP) instead. A laptop has no cloud, so `hostPort` does that job.
- `nodeSelector: ingress-ready`: Traefik must run on the one node whose port 80 your PC forwards to. The label is set in [kind-config.yaml](kind-config.yaml).
- `tolerations`: the control-plane node carries a **taint** (a "keep out" sign for normal pods). A toleration lets Traefik in anyway.
- `isDefaultClass: true`: Kubernetes can have several controllers, each registered as an **IngressClass**. This makes Traefik the one used when an Ingress doesn't pick, and our Ingress also says `ingressClassName: traefik` explicitly.
- `service.spec.type: ClusterIP`: no public IP is needed, since traffic arrives through `hostPort`.

### 6. What Traefik does in this project

- **One address for page and API.** The Angular page and `/api` both come from `mylocal.localhost`. The browser sees one origin, so there are no CORS problems, and that's why the frontend can call a plain `/api/tutorials`.
- **Hiding the internals.** Only `/api` is routed to the backend, so `/actuator/health` isn't reachable from outside. A request for it returns the frontend's `index.html` instead.
- **Hiding the topology.** MongoDB has no route, so nothing outside can reach it.

### 7. Things worth knowing

- **It's an ordinary pod.** Look at it with `kubectl get pods -n traefik` and read its logs with `kubectl logs -n traefik deploy/traefik`. If something is routing wrong, those logs are the first place to look.
- **404 from Traefik means "no rule matched".** Typical causes are a wrong `host`, a wrong path, or a missing Ingress. It does *not* mean your app is broken. A **502/503** means a rule matched but no healthy pod could answer.
- **Only 1 Traefik replica.** That's fine for learning, but it's a single point of failure. In production you'd run several and put a cloud load balancer in front.
- **HTTPS isn't set up.** Port 443 is open (`websecure`), but there's no certificate. Traefik can fetch free Let's Encrypt certificates on a real domain, but that needs a public address, which a local kind cluster doesn't have.
- **Traefik has its own CRDs** (such as `IngressRoute`) with more features than the standard Ingress. This project uses the standard `Ingress` on purpose: it works with any controller, so what you learn here carries over. The newer official successor to Ingress is the **Gateway API**, which Traefik supports too.

### 8. Try it yourself

```bash
# See Traefik and its IngressClass
kubectl get pods -n traefik
kubectl get ingressclass

# Watch it react to routing changes: tail the logs, then edit the host in ingress.yaml and re-apply
kubectl logs -n traefik deploy/traefik -f

# See a "no rule matched" response
curl -i http://wrong.localhost/        # 404 from Traefik
```

## Clean up

```bash
kubectl delete -k k8s/                          # removes the app
kubectl delete pvc data-mongodb-0 -n demo-namespace   # also deletes the MongoDB data
helm uninstall traefik -n traefik
kind delete cluster --name demo-cluster
```

## Notes
- The Secret in `mongodb/secret.yaml` holds learning-only credentials in plain text. Don't commit real passwords.
- MongoDB runs 1 replica on purpose: several plain mongod pods would be separate databases.
- Local development still works unchanged: `mvn spring-boot:run` and `ng serve` (the dev server proxies `/api` to `localhost:8080` via `proxy.conf.json`).
