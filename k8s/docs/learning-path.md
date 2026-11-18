# What to learn next: a Kubernetes learning path

This project already covers the core objects: Pod, Deployment, StatefulSet, Service (ClusterIP and headless), Ingress, ConfigMap, Secret, PersistentVolumeClaim, probes, Kustomize and Helm. The guides next to this file cover [inspecting the cluster](cluster-architecture.md) and [troubleshooting](troubleshooting.md).

This guide lists the topics a beginner should learn next, in a sensible order. Each topic explains **what it is**, **why it matters**, **where it shows up in this project**, and gives an example you can try on `kind-demo-cluster`. Outputs marked *real* come from this cluster.

> **About the examples.** The read-only commands are safe to run any time. Blocks marked **Try it** change the cluster; each one ends with the undo command. New YAML files are only examples and are not part of `k8s/kustomization.yaml`, so `kubectl apply -k k8s/` keeps doing exactly what it does today.

**Contents**

- Part 1, building on what you have: [1. Requests, limits and QoS](#1-resource-requests-limits-and-qos) · [2. Rolling updates](#2-rolling-updates-in-depth) · [3. Labels and selectors](#3-labels-selectors-and-annotations) · [4. Jobs and CronJobs](#4-jobs-and-cronjobs) · [5. Init containers and sidecars](#5-init-containers-and-sidecars) · [6. DaemonSets](#6-daemonsets)
- Part 2, the operating basics: [7. Autoscaling](#7-autoscaling-hpa-and-metrics-server) · [8. NetworkPolicy](#8-networkpolicy) · [9. RBAC](#9-rbac-and-serviceaccounts) · [10. Pod security](#10-pod-security) · [11. Observability](#11-observability-prometheus-and-grafana)
- Part 3, how teams run it: [12. Helm](#12-helm-in-depth) · [13. GitOps](#13-gitops-with-argo-cd) · [14. Gateway API](#14-gateway-api) · [15. StatefulSets done properly](#15-statefulsets-done-properly) · [16. Cluster lifecycle](#16-cluster-lifecycle)
- [Part 4, concepts worth understanding](#part-4-concepts-worth-understanding)
- [Suggested order](#suggested-order-for-this-project)

---

## Part 1: Building on what you have

### 1. Resource requests, limits and QoS

**What it is.** Every container can declare:
- **requests**: what it is guaranteed. The **scheduler** uses only requests to decide which node a pod fits on.
- **limits**: the most it may use. Going over a **CPU** limit makes the container slower (it is throttled). Going over a **memory** limit kills it (`OOMKilled`, exit code 137).

From these, Kubernetes gives each pod a **QoS class**, which decides who gets evicted first when a node runs out of memory:

| QoS class | Rule | Evicted |
|---|---|---|
| `Guaranteed` | Every container has CPU **and** memory requests equal to its limits | Last |
| `Burstable` | At least one request or limit is set, but not Guaranteed | In between |
| `BestEffort` | No requests or limits at all | First |

**In this project.** The backend requests `cpu: 100m, memory: 384Mi` with a `768Mi` memory limit and no CPU limit ([backend/deployment.yaml](../backend/deployment.yaml)). So every pod here is Burstable:

```powershell
kubectl get pods -n demo-namespace -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass
```
*Real:*
```
NAME                        QOS
backend-6458c7bc99-rspb8    Burstable
backend-6458c7bc99-xwd4p    Burstable
frontend-594fdb5667-khp2d   Burstable
frontend-594fdb5667-lc5z8   Burstable
mongo-client                Burstable
mongodb-0                   Burstable
```

**Why the backend has no CPU limit.** A CPU limit throttles the JVM even when the node has idle CPU, which slows down startup and garbage collection. A common practice is: set CPU requests, skip CPU limits, and always set memory limits.

**How to see the effect on a node:**
```powershell
kubectl describe node demo-cluster-worker2 | Select-String -Context 0,6 "Allocated resources"
```
This shows the sum of requests on the node, which is what the scheduler compares with the node's capacity.

**Try it.** Make MongoDB `Guaranteed`, the class you'd want for a database. Edit [mongodb/statefulset.yaml](../mongodb/statefulset.yaml) so requests and limits are equal:
```yaml
resources:
  requests: {cpu: 250m, memory: 1Gi}
  limits:   {cpu: 250m, memory: 1Gi}
```
Apply it with `kubectl apply -k k8s/`, wait for `mongodb-0` to be recreated, and check the QoS column again. Undo by restoring the file and applying again.

---

### 2. Rolling updates in depth

**What it is.** When you change a Deployment's pod template (for example the image), it creates a **new ReplicaSet** and moves pods from the old one to the new one step by step. Two settings control the speed:
- `maxSurge`: how many **extra** pods may exist during the update.
- `maxUnavailable`: how many pods may be **missing** during the update.

**In this project** the defaults are used:
```powershell
kubectl get deploy backend -n demo-namespace -o jsonpath='{.spec.strategy}'
```
*Real:*
```
{"rollingUpdate":{"maxSurge":"25%","maxUnavailable":"25%"},"type":"RollingUpdate"}
```
With 2 replicas, 25% surge rounds **up** to 1 and 25% unavailable rounds **down** to 0. So Kubernetes starts 1 new pod, waits until it is **Ready**, removes 1 old pod, and repeats. The site never has fewer than 2 Ready pods. That's why the probes matter: without a readiness probe, "Ready" would just mean "the process started".

**History and rollback:**
```powershell
kubectl rollout history deploy/backend -n demo-namespace
kubectl get rs -n demo-namespace
```
*Real:*
```
REVISION  CHANGE-CAUSE
1         <none>

NAME                  DESIRED   CURRENT   READY
backend-6458c7bc99    2         2         2
frontend-594fdb5667   2         2         2
```
Only revision 1 exists so far. Old ReplicaSets are kept (scaled to 0) up to `revisionHistoryLimit`, 10 here, and that's what makes `rollout undo` possible.

**Try it.** Release a new backend version and watch the swap:
```powershell
docker build -t belajar-backend:1.0.1 spring-boot-data-mongodb
kind load docker-image belajar-backend:1.0.1 --name demo-cluster

kubectl set image deploy/backend backend=belajar-backend:1.0.1 -n demo-namespace
kubectl annotate deploy/backend kubernetes.io/change-cause="backend 1.0.1" -n demo-namespace
kubectl get rs -n demo-namespace -l app=backend -w      # new RS goes 0→1→2, old RS goes 2→1→0

kubectl rollout history deploy/backend -n demo-namespace   # now shows revision 2 with its cause
kubectl rollout undo deploy/backend -n demo-namespace      # back to 1.0.0
```

**Related settings worth knowing:**
- `minReadySeconds: 10`: a new pod must stay Ready 10 seconds before it counts. This catches apps that crash right after starting.
- `progressDeadlineSeconds` (default 600): after that, `rollout status` reports the rollout as failed.
- `strategy: Recreate`: kill all old pods first, then start new ones. There's downtime, but it's needed when two versions must never run at the same time.
- `kubectl rollout pause` / `resume`: make several changes and roll them out once.

---

### 3. Labels, selectors and annotations

**What it is.**
- A **label** is a key/value tag on any object, for example `app: backend`. On its own it does nothing.
- A **selector** is a query: "all objects with label `app=backend`".
- **Annotations** are also key/value data, but for information that tools read, not for selecting.

Selectors are how objects find each other in Kubernetes, by label, not by name. So there are two roles to keep apart:
- **`labels:`** means "this is what *I* am tagged with".
- **`selector:`** / **`matchLabels:`** means "these are the *other* objects I'm looking for".

#### The three labels in backend/deployment.yaml

[backend/deployment.yaml](../backend/deployment.yaml) has `app: backend` in **three** places, and each one has a different job:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:              # ① the Deployment's own label
    app: backend
spec:
  replicas: 2
  selector:
    matchLabels:       # ② which pods this Deployment owns
      app: backend
  template:
    metadata:
      labels:          # ③ the label every new pod gets
        app: backend
    spec:
      containers: ...
```

| # | Where | Labels what | Who reads it | What happens if it's wrong |
|---|---|---|---|---|
| ① | `metadata.labels` | The **Deployment object** | Only you, via `kubectl get deploy -l app=backend` | Nothing breaks. It's just a tag for searching. |
| ② | `spec.selector.matchLabels` | Nothing; it's a **query** | The Deployment and its ReplicaSet, to count "my pods" | Kubernetes rejects the file if ② doesn't match ③ |
| ③ | `spec.template.metadata.labels` | Each **pod** created from the template | The Deployment (②), the **Service**, and later HPA, PDB and NetworkPolicy | The Service finds no pods, so the site returns 503 |

**① The Deployment's own label** is an "about me" tag on the Deployment itself. No Kubernetes component depends on it; you could delete it and the app would keep working. It lets you search:

```powershell
kubectl get deploy -n demo-namespace -l app=backend
kubectl get all -n demo-namespace -l app=backend        # everything tagged backend
```

**③ The pod template's label.** `template` is the **blueprint** for pods, and `template.metadata.labels` is stamped onto every pod created from it. This is the label that matters most, because the other objects select on it. The live pods:

*Real:*
```
pod/backend-6458c7bc99-rspb8    app=backend,pod-template-hash=6458c7bc99
pod/backend-6458c7bc99-xwd4p    app=backend,pod-template-hash=6458c7bc99
```
`pod-template-hash` was added by Kubernetes; see [Labels Kubernetes adds itself](#labels-kubernetes-adds-itself).

**② The selector** tells the Deployment: "the pods with `app=backend` are mine". The Deployment (through its ReplicaSet) keeps counting matching pods and creates or deletes pods until there are exactly 2. Rules:
- ② **must match** ③, or the API server rejects the file with `selector does not match template labels`. The pods it creates have to be pods it can find.
- ② **can't be changed** after the Deployment exists. To change it, you delete and recreate the Deployment.
- ② can be **narrower** than ③: the template can have extra labels that the selector ignores.

**Why ② and ③ are separate when they look identical.** The selector is the stable identity ("which pods are mine, forever"). The template labels can carry more, and can change between versions:

```yaml
selector:
  matchLabels:
    app: backend                # stable: never changes
template:
  metadata:
    labels:
      app: backend
      version: "1.0.1"          # changes every release; the selector doesn't care
```

#### How they connect: the full chain for the backend

```
              selects app=backend               selects app=backend
Deployment ─────────────────► ReplicaSet ───────────────────────► Pods (app=backend)
  backend      (owns it)        backend-6458c7bc99   (creates them)    ▲
                                                                       │ selects app=backend
                                                     Service backend-service
                                                                       │ writes the pod IPs into
                                                                       ▼
                                                     EndpointSlice backend-service-tzrjw
                                                        10.244.1.2, 10.244.2.8
                                                                       ▲
                                                     Ingress demo-ingress ── by NAME ──┘
                                                       (/api → backend-service)
```

**The Service never talks to the Deployment.** They don't know about each other; both independently look for pods labelled `app=backend`. If you created a pod by hand with `app=backend`, the Service would send traffic to it too.

The **Ingress** is the exception: it points to the Service by **name** (`backend-service`), not by label. That's why renaming the Service breaks the Ingress.

#### Labels in the Service

[backend/service.yaml](../backend/service.yaml) has two blocks, with the same two roles:

```yaml
metadata:
  name: backend-service
  labels:            # the Service's own tag (like ①)
    app: backend
spec:
  selector:          # which pods receive the traffic (like ②)
    app: backend
```

| Field | Purpose | If wrong |
|---|---|---|
| `metadata.labels` | Tag for searching | Nothing breaks |
| `spec.selector` | Picks the pods that receive traffic | **No endpoints**, so Traefik returns 503 |

Two differences from a Deployment:
- A Service's selector is written **directly** as `selector: {app: backend}`. It has no `matchLabels`, because Services only support simple equality matching.
- A Service's selector **can** be changed at any time, and the endpoints update within seconds.

The result of the Service selector:
```powershell
kubectl get endpointslices -n demo-namespace
```
*Real:*
```
backend-service-tzrjw    IPv4   8080    10.244.1.2,10.244.2.8   <- exactly the two app=backend pods
```

#### Labels in ConfigMaps and Secrets

[backend/configmap.yaml](../backend/configmap.yaml), [frontend/configmap.yaml](../frontend/configmap.yaml) and [mongodb/secret.yaml](../mongodb/secret.yaml) only have `metadata.labels`:

```yaml
kind: ConfigMap
metadata:
  name: backend-config
  labels:
    app: backend      # just a tag
```

They have **no selector**, because ConfigMaps and Secrets don't look for anything. They're **pulled by name** by the pod:

```yaml
# in backend/deployment.yaml
envFrom:
  - configMapRef:
      name: backend-config          # by NAME, not label
env:
  - name: SPRING_DATA_MONGODB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mongodb-secret        # by NAME
        key: MONGO_INITDB_ROOT_PASSWORD
```

So the label `app: backend` on the ConfigMap is purely for grouping. It lets you find "everything that belongs to the backend" in one command:
```powershell
kubectl get deploy,svc,cm,pods -n demo-namespace -l app=backend
```
Removing that label changes nothing about how the app works. Changing the ConfigMap's **name** would break the pods (`CreateContainerConfigError`, see [troubleshooting Failure 4](troubleshooting.md#failure-4-createcontainerconfigerror)).

The Secret is labelled `app: mongodb`, not `app: backend`, even though the backend reads it too. It belongs to MongoDB, and the backend borrows it. Labels describe what an object *is*, not who uses it.

#### Labels in the StatefulSet and its PVC

[mongodb/statefulset.yaml](../mongodb/statefulset.yaml) has the same three labels as the Deployment, with the same meaning:

```yaml
metadata:
  labels: {app: mongodb}          # ① the StatefulSet's own tag
spec:
  selector:
    matchLabels: {app: mongodb}   # ② which pods are mine
  template:
    metadata:
      labels: {app: mongodb}      # ③ label on mongodb-0
```

The headless Service [mongodb/service.yaml](../mongodb/service.yaml) selects `app: mongodb`, which is how the name `mongodb` resolves to `mongodb-0`'s IP.

The PVC `data-mongodb-0` also shows `app=mongodb`, although no label is written for it. The StatefulSet controller copies its selector labels onto the PVCs it creates from `volumeClaimTemplates`. That's why this works:
```powershell
kubectl get statefulset,pod,pvc -n demo-namespace -l app=mongodb
```

#### Labels Kubernetes adds itself

The live objects have labels nobody wrote. Don't remove them; controllers rely on them.

| Label | On | Added by | Purpose |
|---|---|---|---|
| `pod-template-hash=6458c7bc99` | ReplicaSet and its pods | Deployment controller | A hash of the pod template. During an update, the old and new ReplicaSet each find **only their own** pods, even though both have `app=backend`. It's also the middle part of the pod name `backend-6458c7bc99-rspb8`. |
| `controller-revision-hash=mongodb-cd9f5f9fb` | StatefulSet pods | StatefulSet controller | Which version of the template the pod was made from |
| `statefulset.kubernetes.io/pod-name=mongodb-0` | StatefulSet pods | StatefulSet controller | Lets you target one specific pod, for example with its own Service |
| `apps.kubernetes.io/pod-index=0` | StatefulSet pods | StatefulSet controller | The pod's ordinal number |
| `kubernetes.io/service-name=backend-service` | EndpointSlice | EndpointSlice controller | Ties the slice to its Service |
| `kubernetes.io/metadata.name=demo-namespace` | Namespace | API server | Lets NetworkPolicies select namespaces by name ([topic 8](#8-networkpolicy)) |
| `node-role.kubernetes.io/control-plane` | Node | kubeadm (kind) | Marks the control-plane node ([cluster-architecture.md](cluster-architecture.md#3-how-kubernetes-knows-a-nodes-role-labels)) |

```powershell
kubectl get rs,pods -n demo-namespace --show-labels
```

#### Labels on nodes

[kind-config.yaml](../kind-config.yaml) puts a label on a **node**:
```yaml
- role: control-plane
  labels:
    ingress-ready: "true"
```
[traefik-values.yaml](../traefik-values.yaml) selects it with a `nodeSelector`. It's the same pattern, except that here a pod uses a selector to choose a node:
```yaml
nodeSelector:
  ingress-ready: "true"     # run Traefik only on nodes with this label
```

#### Annotations: the other kind of metadata

Annotations look like labels (key/value under `metadata`), but **you can't select on them**. They hold extra information for tools or people:

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "backend 1.0.1"        # shown by kubectl rollout history
    alb.ingress.kubernetes.io/scheme: internet-facing  # read by the AWS load balancer controller
    description: "Spring Boot REST API for tutorials"  # just for people
```

| | Labels | Annotations |
|---|---|---|
| Used for | Selecting and grouping objects | Extra information for tools and people |
| Can be selected with `-l` or `selector` | Yes | No |
| Value limits | Short (≤ 63 chars: letters, digits, `-`, `_`, `.`) | Any text, even large JSON |
| Example in this project | `app: backend` | `kubectl.kubernetes.io/last-applied-configuration` (added by `kubectl apply`), `kubectl.kubernetes.io/restartedAt` (added by `kubectl rollout restart`) |

```powershell
kubectl get deploy backend -n demo-namespace -o jsonpath='{.metadata.annotations}'
```

#### Recommended labels

Larger projects use a standard set of labels that tools like Helm and dashboards understand:
```yaml
labels:
  app.kubernetes.io/name: backend
  app.kubernetes.io/part-of: belajar
  app.kubernetes.io/version: "1.0.0"
  app.kubernetes.io/managed-by: kustomize
```
Kustomize can add labels to every object with a `labels:` entry in [kustomization.yaml](../kustomization.yaml). Be careful when adding them to an existing app: if they end up in a Deployment's **selector**, the apply fails, because a selector can't change.

#### Summary: who finds whom, and how

| From | To | How |
|---|---|---|
| Deployment / StatefulSet | its pods | **label selector** (`spec.selector.matchLabels`) |
| ReplicaSet | its pods | label selector + `pod-template-hash` |
| Service | pods | **label selector** (`spec.selector`) |
| HPA, PDB, NetworkPolicy | pods | label selector |
| Traefik (via `nodeSelector`) | node | node label `ingress-ready=true` |
| Ingress | Service | **name** (`backend-service`) |
| Pod | ConfigMap / Secret | **name** (`backend-config`, `mongodb-secret`) |
| Pod | PVC | **name** (StatefulSet: `data-mongodb-0`, from the template) |
| You | anything | label selector with `kubectl -l`, for convenience |

The rule of thumb: Kubernetes connects to **pods** with labels, because pods come and go and their names change. It connects to **stable objects** like Services, ConfigMaps and Secrets by **name**.

**Try it.** Take a pod out of service without killing it, a classic debugging trick that also shows the whole chain is just labels:
```powershell
kubectl label pod <backend-pod> -n demo-namespace app=debug --overwrite
kubectl get pods -n demo-namespace --show-labels    # the Deployment created a replacement: it saw only 1 app=backend pod
kubectl get endpointslices -n demo-namespace        # the relabelled pod is no longer behind backend-service
```
The relabelled pod keeps running, but nothing owns it or sends traffic to it, so you can `exec` into it or read its logs in peace. Delete it when you're done: `kubectl delete pod <backend-pod> -n demo-namespace`.

---

### 4. Jobs and CronJobs

**What it is.** Deployments run servers that never finish. A **Job** runs a pod **until it completes successfully** (retrying on failure). A **CronJob** creates a Job on a schedule. Typical uses are database backups, migrations, reports and clean-ups.

**For this project:** a nightly MongoDB backup is a real, useful job. Save as `k8s/examples/mongodb-backup-cronjob.yaml`:

```yaml
# Storage for the backup files
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-backup
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mongodb-backup
spec:
  schedule: "0 2 * * *"            # every day at 02:00 (cluster time, UTC)
  concurrencyPolicy: Forbid        # don't start a new backup if the last one is still running
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2              # retry a failed backup twice
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: mongodump
              image: mongo:7       # same image as the database, so the tools match
              envFrom:
                - secretRef:
                    name: mongodb-secret
              command: ["sh", "-c"]
              args:
                - >
                  mongodump --host mongodb --authenticationDatabase admin
                  --username "$MONGO_INITDB_ROOT_USERNAME" --password "$MONGO_INITDB_ROOT_PASSWORD"
                  --db belajar_db --gzip --archive=/backup/belajar_db-$(date +%Y%m%d-%H%M%S).gz
                  && ls -lh /backup
              volumeMounts:
                - name: backup
                  mountPath: /backup
          volumes:
            - name: backup
              persistentVolumeClaim:
                claimName: mongodb-backup
```

**Try it.** Apply it, then trigger a backup now instead of waiting for 02:00:
```powershell
kubectl apply -f k8s/examples/mongodb-backup-cronjob.yaml -n demo-namespace
kubectl get cronjob -n demo-namespace

kubectl create job backup-now --from=cronjob/mongodb-backup -n demo-namespace
kubectl get jobs,pods -n demo-namespace -l job-name=backup-now -w     # Completed
kubectl logs job/backup-now -n demo-namespace                        # shows the .gz file
```
Restoring works the same way with a one-off Job running `mongorestore --gzip --archive=/backup/<file>`.

Undo:
```powershell
kubectl delete -f k8s/examples/mongodb-backup-cronjob.yaml -n demo-namespace
kubectl delete job backup-now -n demo-namespace
```

**Things to know**
- A Job's pod needs `restartPolicy: OnFailure` or `Never`; `Always` isn't allowed.
- Finished Jobs stay around so you can read their logs. `ttlSecondsAfterFinished` deletes them automatically.
- A backup on the same disk, in the same cluster, is not a real backup. In production it would be copied to object storage such as S3.

---

### 5. Init containers and sidecars

**What it is.**
- An **init container** runs **before** the app containers and must finish successfully first. Use it to wait for a dependency, run a migration or prepare files.
- A **sidecar** runs **next to** the app for the pod's whole life. Use it for a log shipper, a proxy or a config reloader. Since Kubernetes 1.33 a sidecar is declared as an init container with `restartPolicy: Always`, so it starts first and stops last.

**A real problem in this project.** The backend's startup log contains:
```
com.mongodb.MongoSocketException: mongodb: Temporary failure in name resolution
```
The backend started before `mongodb-0` was Ready, and the MongoDB driver retried until it connected ([troubleshooting Failure 9](troubleshooting.md#failure-9-dns-doesnt-resolve)). It works, but an init container makes the order explicit. Add this under `spec.template.spec` in [backend/deployment.yaml](../backend/deployment.yaml):

```yaml
initContainers:
  - name: wait-for-mongodb
    image: busybox:1.37
    command: ["sh", "-c"]
    args:
      - until nslookup mongodb.demo-namespace.svc.cluster.local; do
          echo "waiting for mongodb..."; sleep 2;
        done
    resources:
      requests: {cpu: 10m, memory: 16Mi}
      limits: {memory: 32Mi}
```

**Try it.**
```powershell
kubectl apply -k k8s/
kubectl scale statefulset mongodb --replicas=0 -n demo-namespace
kubectl rollout restart deploy/backend -n demo-namespace
kubectl get pods -n demo-namespace -l app=backend -w     # new pods wait in Init:0/1
kubectl logs <new-backend-pod> -c wait-for-mongodb -n demo-namespace   # "waiting for mongodb..."
kubectl scale statefulset mongodb --replicas=1 -n demo-namespace       # the backends then start
```
To undo, remove the `initContainers` block and apply again.

**When not to use it.** The app should still handle the database going away **later**. An init container only helps at startup. Here the MongoDB driver's retries already cover both cases, so this is about clarity, not a bug fix.

**Sidecar example** (a shape to recognise, not needed here): an nginx sidecar that writes access logs to a shared `emptyDir` volume, read by a log-shipping sidecar.
```yaml
initContainers:
  - name: log-shipper
    image: busybox:1.37
    restartPolicy: Always            # this makes it a sidecar
    command: ["sh", "-c", "tail -F /logs/access.log"]
    volumeMounts: [{name: logs, mountPath: /logs}]
```

---

### 6. DaemonSets

**What it is.** A DaemonSet runs **exactly one pod on every node** (or on every node matching a selector). When a node joins, it gets the pod automatically. Typical uses are network plugins, log collectors and monitoring agents.

**In this project** you already have two, installed by kind:
```powershell
kubectl get ds -A
```
*Real:*
```
NAMESPACE     NAME         DESIRED   CURRENT   READY   NODE SELECTOR            AGE
kube-system   kindnet      3         3         3       kubernetes.io/os=linux   22h
kube-system   kube-proxy   3         3         3       kubernetes.io/os=linux   22h
```
DESIRED is 3 because there are 3 nodes. They even run on the control plane, because DaemonSets for system components tolerate its taint.

```powershell
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide   # one per node
kubectl describe ds kube-proxy -n kube-system                   # see its Tolerations
```

**Try it.** A tiny DaemonSet that shows each node's name. Save as `k8s/examples/node-hello-daemonset.yaml`:
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-hello
spec:
  selector:
    matchLabels: {app: node-hello}
  template:
    metadata:
      labels: {app: node-hello}
    spec:
      containers:
        - name: hello
          image: busybox:1.37
          env:
            - name: NODE_NAME
              valueFrom: {fieldRef: {fieldPath: spec.nodeName}}
          command: ["sh", "-c", "while true; do echo running on $NODE_NAME; sleep 30; done"]
          resources:
            requests: {cpu: 5m, memory: 8Mi}
            limits: {memory: 16Mi}
```
```powershell
kubectl apply -f k8s/examples/node-hello-daemonset.yaml -n demo-namespace
kubectl get pods -n demo-namespace -l app=node-hello -o wide    # 2 pods: one per worker
kubectl logs -n demo-namespace -l app=node-hello --prefix
kubectl delete -f k8s/examples/node-hello-daemonset.yaml -n demo-namespace   # undo
```
You get 2 pods, not 3: this DaemonSet has no toleration, so the control plane's taint keeps it off. Adding the toleration from [traefik-values.yaml](../traefik-values.yaml) would make it 3.

---

## Part 2: The operating basics

### 7. Autoscaling (HPA) and metrics-server

**What it is.** A **HorizontalPodAutoscaler** changes a Deployment's replica count based on a metric, usually CPU usage measured **as a percentage of the CPU request**. It needs **metrics-server**, which collects CPU and memory from each kubelet. kind doesn't include it:
```powershell
kubectl top nodes
# real: error: Metrics API not available
```

**Step 1: install metrics-server.** kind's kubelets use self-signed certificates, so metrics-server needs `--kubelet-insecure-tls`. That's fine for kind and never for production:
```powershell
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm install metrics-server metrics-server/metrics-server -n kube-system --set "args={--kubelet-insecure-tls}"
kubectl top nodes                      # works after about a minute
kubectl top pods -n demo-namespace
```

**Step 2: an HPA for the backend.** Save as `k8s/examples/backend-hpa.yaml`:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50     # 50% of the 100m request = 50m per pod
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60 # wait 1 min of low load before removing pods (default is 5 min)
```

**Step 3: create load and watch it scale.**
```powershell
kubectl apply -f k8s/examples/backend-hpa.yaml -n demo-namespace

# a busy loop calling the API from inside the cluster
kubectl run load -n demo-namespace --image=busybox:1.37 --restart=Never -- sh -c "while true; do wget -q -O- http://backend-service:8080/api/tutorials > /dev/null; done"

kubectl get hpa backend -n demo-namespace -w      # TARGETS rises above 50%, REPLICAS goes up
kubectl get pods -n demo-namespace -l app=backend -w
```
Stop the load, and after about a minute the replicas drop back to 2:
```powershell
kubectl delete pod load -n demo-namespace
kubectl delete -f k8s/examples/backend-hpa.yaml -n demo-namespace    # undo
```

**Things to know**
- **The request is the yardstick.** The backend requests only 100m, so "50%" is just 50m. A request that is too low makes the HPA scale too eagerly, and one that is too high makes it never scale.
- **HPA vs. `replicas:` in git.** [backend/deployment.yaml](../backend/deployment.yaml) says `replicas: 2`, so every `kubectl apply -k k8s/` resets the count to 2, fighting the HPA. With an HPA, remove `replicas` from the Deployment ([production.md step 8](production.md#step-8-create-a-kustomize-overlay-for-eks) does this).
- New pods need **room on the nodes**. In the cloud, a node autoscaler (Karpenter, Cluster Autoscaler) adds nodes. In kind, the node count is fixed.
- The **VerticalPodAutoscaler** (VPA) suggests better requests instead of adding pods. KEDA scales on queue length or other events.

---

### 8. NetworkPolicy

**What it is.** By default, **every pod can reach every pod** in every namespace. A NetworkPolicy is a firewall rule for pods: once any policy selects a pod, only the traffic the policies allow reaches it.

**In this project, today:** `mongo-client`, or any pod in any namespace, can connect to MongoDB on port 27017, and anyone inside the cluster can call the backend directly. Only these paths are actually needed:

```
traefik (namespace traefik) ──► frontend :80
traefik (namespace traefik) ──► backend  :8080
backend                      ──► mongodb  :27017
(mongo-client ──► mongodb, for debugging)
```

**Important for kind.** NetworkPolicies are enforced by the network plugin (CNI). kind's default `kindnet` **does not enforce them**: you can apply policies, but nothing is blocked. To practise, create a separate cluster with Calico:

```yaml
# k8s/examples/kind-calico.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true        # no kindnet
  podSubnet: 192.168.0.0/16      # Calico's default pod range
nodes:
  - role: control-plane
  - role: worker
```
```powershell
kind create cluster --name netpol-lab --config k8s/examples/kind-calico.yaml
# install Calico, following https://docs.tigera.io/calico/latest/getting-started/kubernetes/kind
```
Deploy the app there the same way as in [README.md](../README.md#first-time-setup-run-from-the-repo-root), using `--name netpol-lab` for `kind load`.

**The policies.** Save as `k8s/examples/networkpolicies.yaml`:
```yaml
# 1. Nothing gets in unless allowed below
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes: [Ingress]
---
# 2. Traefik → frontend and backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-traefik
spec:
  podSelector:
    matchExpressions:
      - {key: app, operator: In, values: [frontend, backend]}
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: traefik   # every namespace has this label automatically
---
# 3. backend (and the debug pod) → mongodb
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-to-mongodb
spec:
  podSelector: {matchLabels: {app: mongodb}}
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: {matchLabels: {app: backend}}
        - podSelector: {matchLabels: {app: mongo-client}}
      ports:
        - port: 27017
```

**Test that it works** (on the Calico cluster):
```powershell
kubectl apply -f k8s/examples/networkpolicies.yaml -n demo-namespace

# allowed: the site still works
curl.exe http://mylocal.localhost/api/tutorials

# blocked: a random pod can't reach MongoDB any more
kubectl run probe -n demo-namespace --rm -it --image=busybox:1.37 --restart=Never -- nc -zv -w 3 mongodb 27017
# times out

kubectl delete -f k8s/examples/networkpolicies.yaml -n demo-namespace   # undo
kind delete cluster --name netpol-lab                                    # when finished
```

**Things to know**
- Policies only **add** permissions. There is no "deny" rule other than "not allowed by any policy".
- These examples only restrict **incoming** traffic. Restricting **outgoing** traffic (`Egress`) also needs a rule that allows DNS (UDP/TCP 53 to kube-dns), or everything breaks.
- On EKS, the VPC CNI can enforce policies ([production.md step 10](production.md#step-10-lock-down-pod-to-pod-traffic-networkpolicy)).

---

### 9. RBAC and ServiceAccounts

**What it is.** RBAC (role-based access control) decides **who can do what** with the API:
- A **Role** lists allowed actions (verbs such as `get`, `list`, `create`, `delete`) on resources in **one namespace**. A **ClusterRole** does the same cluster-wide.
- A **RoleBinding** (or ClusterRoleBinding) gives a Role to a **subject**: a user, a group, or a **ServiceAccount**.
- A **ServiceAccount** is an identity for **pods and programs**. Every pod runs as one; if you don't pick one, it's `default`.

**In this project.** You use kind's admin credentials, so `kubectl` can do anything. The pods run as the `default` ServiceAccount, which can do almost nothing:
```powershell
kubectl get serviceaccounts -n demo-namespace
kubectl auth can-i list pods -n demo-namespace --as=system:serviceaccount:demo-namespace:default
```
*Real:*
```
NAME      AGE
default   22h
no
```
That's good: your backend has no reason to talk to the Kubernetes API, so it shouldn't be able to.

**Try it.** A read-only "viewer" for `demo-namespace`. Save as `k8s/examples/viewer-rbac.yaml`:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: viewer
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: read-only
rules:
  - apiGroups: ["", "apps"]          # "" = core (pods, services...), apps = deployments...
    resources: ["pods", "pods/log", "services", "endpoints", "configmaps", "deployments", "replicasets", "statefulsets"]
    verbs: ["get", "list", "watch"]
  # note: no "secrets", so the viewer can't read the MongoDB password
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: viewer-read-only
subjects:
  - kind: ServiceAccount
    name: viewer
    namespace: demo-namespace
roleRef:
  kind: Role
  name: read-only
  apiGroup: rbac.authorization.k8s.io
```
```powershell
kubectl apply -f k8s/examples/viewer-rbac.yaml -n demo-namespace

$as = "--as=system:serviceaccount:demo-namespace:viewer"
kubectl auth can-i list pods      -n demo-namespace $as   # yes
kubectl auth can-i delete pods    -n demo-namespace $as   # no
kubectl auth can-i get secrets    -n demo-namespace $as   # no
kubectl auth can-i list pods      -n kube-system   $as    # no (a Role covers one namespace)
kubectl get pods -n demo-namespace $as                   # works
kubectl delete pod mongo-client -n demo-namespace $as    # Forbidden

kubectl auth can-i --list -n demo-namespace $as          # everything it may do
```
Undo with `kubectl delete -f k8s/examples/viewer-rbac.yaml -n demo-namespace`.

**Things to know**
- Kubernetes ships ready-made ClusterRoles: `view`, `edit`, `admin` and `cluster-admin`. Binding `view` with a RoleBinding gives read-only access to one namespace without writing a Role.
- `kubectl create token viewer -n demo-namespace` prints a short-lived token for that ServiceAccount, for example for a dashboard or CI.
- If a pod doesn't need the API, set `automountServiceAccountToken: false` in its spec, so no token is mounted into it.

---

### 10. Pod security

**What it is.** Two parts:
- The **securityContext** in a pod spec restricts what a container may do: run as a non-root user, no privilege escalation, drop Linux capabilities, read-only root filesystem, a seccomp profile.
- **Pod Security Admission** enforces a standard per namespace with a label. The levels are `privileged` (anything goes), `baseline` (blocks the obviously dangerous) and `restricted` (best practice). Each can be set to `enforce` (reject), `warn` (warn on apply) or `audit` (log).

**In this project:** the backend image already runs as a non-root user (uid 10001, from its [Dockerfile](../../spring-boot-data-mongodb/Dockerfile)), but the manifests don't declare any securityContext, and nginx and mongo run as root.

**See what `restricted` would reject, without changing anything** (`--dry-run=server` only reports):
```powershell
kubectl label --dry-run=server --overwrite ns demo-namespace pod-security.kubernetes.io/enforce=restricted
```
Kubernetes prints a warning for each existing pod that would violate the standard, for example `allowPrivilegeEscalation != false`, `runAsNonRoot != true`, `seccompProfile`.

**What a restricted-compliant backend container looks like:**
```yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        seccompProfile: {type: RuntimeDefault}
      containers:
        - name: backend
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: {drop: [ALL]}
          volumeMounts:
            - {name: tmp, mountPath: /tmp}   # Spring Boot / Tomcat need a writable /tmp
      volumes:
        - {name: tmp, emptyDir: {}}
```
The frontend would need the `nginxinc/nginx-unprivileged` image (port 8080 instead of 80), because the standard nginx image starts as root. The Service's `targetPort` then changes to 8080.

**A safe way to start** is warn mode, which never blocks:
```powershell
kubectl label ns demo-namespace pod-security.kubernetes.io/warn=restricted
kubectl apply -k k8s/                                         # prints warnings, applies anyway
kubectl label ns demo-namespace pod-security.kubernetes.io/warn-   # undo (trailing "-" removes the label)
```

---

### 11. Observability: Prometheus and Grafana

**What it is.**
- **Metrics:** numbers over time, such as request rate, error rate, latency, JVM memory and pod CPU. **Prometheus** scrapes and stores them, and **Grafana** draws dashboards.
- **Logs:** what `kubectl logs` shows. They disappear with the pod unless shipped somewhere (Loki, CloudWatch).
- **Traces:** the path of one request through several services (OpenTelemetry).

**In this project** you can only look at logs and `kubectl describe` today. The backend already has Spring Boot Actuator, so adding metrics is a small change.

**Step 1: the monitoring stack.** One Helm chart installs Prometheus, Grafana, Alertmanager, node and Kubernetes metrics, and ready-made dashboards:
```powershell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

kubectl get pods -n monitoring -w        # wait until all are Running
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```
Open http://localhost:3000. The user is `admin`; get the password with:
```powershell
$p = kubectl get secret monitoring-grafana -n monitoring -o jsonpath='{.data.admin-password}'
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($p))
```
Look at **Dashboards → Kubernetes / Compute Resources / Namespace (Pods)** and pick `demo-namespace`. It shows CPU and memory for your backend, frontend and mongodb pods compared with their requests and limits, which ties back to [topic 1](#1-resource-requests-limits-and-qos).

**Step 2: application metrics from the backend.**
1. Add the dependency to [pom.xml](../../spring-boot-data-mongodb/pom.xml):
   ```xml
   <dependency>
     <groupId>io.micrometer</groupId>
     <artifactId>micrometer-registry-prometheus</artifactId>
   </dependency>
   ```
2. Expose the endpoint in [application.properties](../../spring-boot-data-mongodb/src/main/resources/application.properties):
   ```properties
   management.endpoints.web.exposure.include=health,prometheus
   ```
3. Rebuild the image as `1.0.1`, load it into kind and deploy it (see [topic 2](#2-rolling-updates-in-depth)). Check it with `kubectl port-forward svc/backend-service 8080:8080 -n demo-namespace` and http://localhost:8080/actuator/prometheus.
4. Tell Prometheus to scrape it with a **ServiceMonitor**, a custom resource that comes with the chart:
   ```yaml
   apiVersion: monitoring.coreos.com/v1
   kind: ServiceMonitor
   metadata:
     name: backend
     namespace: demo-namespace
     labels:
       release: monitoring          # the chart only picks up ServiceMonitors with its release label
   spec:
     selector:
       matchLabels: {app: backend}  # the label on backend-service
     endpoints:
       - port: http                 # the Service's port name
         path: /actuator/prometheus
         interval: 30s
   ```
In Grafana's **Explore**, try `http_server_requests_seconds_count{namespace="demo-namespace"}` and `jvm_memory_used_bytes`.

Undo: `helm uninstall monitoring -n monitoring`, then `kubectl delete ns monitoring`. The stack uses a fair amount of memory; uninstall it when you aren't using it.

---

## Part 3: How teams run it

### 12. Helm in depth

**What it is.** Helm is a package manager for Kubernetes. A **chart** is a folder of templated YAML plus a `values.yaml` file of settings. A **release** is one installed copy of a chart. Helm keeps a history of each release, so you can upgrade and roll back.

**In this project** you used someone else's chart for Traefik:
```powershell
helm list -A                                  # releases: traefik
helm get values traefik -n traefik            # the values you gave it (from traefik-values.yaml)
helm history traefik -n traefik               # revisions of the release
helm template traefik traefik/traefik -f k8s/traefik-values.yaml | Select-Object -First 40   # render without installing
```

**Try it: write a chart for this app.**
```powershell
helm create k8s/examples/belajar-chart     # generates a starter chart
```
```
belajar-chart/
├── Chart.yaml           # name, version
├── values.yaml          # settings: image.tag, replicaCount, resources...
└── templates/           # YAML with {{ .Values.xxx }} placeholders
```
Replace the generated templates with your backend's Deployment and Service, and turn the parts that change into values:
```yaml
# templates/backend-deployment.yaml (excerpt)
spec:
  replicas: {{ .Values.backend.replicas }}
  template:
    spec:
      containers:
        - name: backend
          image: "{{ .Values.backend.image }}:{{ .Values.backend.tag }}"
```
```yaml
# values.yaml
backend:
  image: belajar-backend
  tag: "1.0.0"
  replicas: 2
```
```powershell
helm template k8s/examples/belajar-chart                     # check the output
helm install belajar k8s/examples/belajar-chart -n helm-test --create-namespace
helm upgrade belajar k8s/examples/belajar-chart -n helm-test --set backend.replicas=3
helm rollback belajar 1 -n helm-test
helm uninstall belajar -n helm-test; kubectl delete ns helm-test    # undo
```

**Helm or Kustomize?**

| | Kustomize (this project) | Helm |
|---|---|---|
| Approach | Plain YAML plus patches | Templates plus values |
| Good for | Your own apps, small differences between environments | Sharing an app with others, many settings |
| Built into kubectl | Yes (`-k`) | No, separate tool |
| Release history and rollback | No (use git) | Yes |

Many teams use both: Helm for third-party software (Traefik, Prometheus) and Kustomize for their own apps.

---

### 13. GitOps with Argo CD

**What it is.** In GitOps, **git is the source of truth**. A controller in the cluster (Argo CD or Flux) keeps comparing the cluster with a folder in git and applies any difference. Nobody runs `kubectl apply` by hand; a change is a commit, and a rollback is a `git revert`.

**In this project** you deploy with `kubectl apply -k k8s/`. With Argo CD, the cluster would follow the `k8s/` folder of your GitHub repo by itself.

**Try it:**
```powershell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w

# the initial admin password
$p = kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}'
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($p))

kubectl port-forward svc/argocd-server 8443:443 -n argocd     # https://localhost:8443, user admin
```
Then tell it what to follow. Save as `k8s/examples/argocd-app.yaml`, with your repo's URL:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: belajar
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<owner>/spring-mongodb-angular-k8s.git
    targetRevision: main
    path: k8s                      # Argo CD sees kustomization.yaml and runs Kustomize
  destination:
    server: https://kubernetes.default.svc
    namespace: demo-namespace
  syncPolicy:
    automated:
      prune: true                  # delete objects that were removed from git
      selfHeal: true               # undo manual changes made in the cluster
```
```powershell
kubectl apply -f k8s/examples/argocd-app.yaml
```

**See it work:**
- Change `replicas: 2` to `3` in [frontend/deployment.yaml](../frontend/deployment.yaml), commit and push. Within about 3 minutes Argo CD shows "OutOfSync", then syncs, and a third frontend pod appears.
- Run `kubectl scale deploy/frontend --replicas=1 -n demo-namespace`. Self-heal puts it back to what git says.

**Things to know**
- The images still have to reach the nodes. In kind that's still `kind load`; in real setups CI pushes them to a registry, and the commit only changes the tag ([production.md step 13](production.md#step-13-automate-builds-and-deploys-ci--gitops)).
- `k8s/mongodb/secret.yaml` would be synced from git too. That's one more reason to move secrets out of git (Sealed Secrets, External Secrets) before using GitOps for real.

Undo: `kubectl delete -f k8s/examples/argocd-app.yaml` (with `prune` on, this can delete the app's objects, so re-run `kubectl apply -k k8s/` afterwards), then `kubectl delete ns argocd`.

---

### 14. Gateway API

**What it is.** The successor to Ingress. Ingress can do host and path routing, and everything else (redirects, header matching, traffic splitting, timeouts) goes into controller-specific annotations. Gateway API makes these first-class and splits the roles:
- **GatewayClass**: which controller (for example Traefik). It's set up by the cluster provider.
- **Gateway**: a listener, such as port 80 for some hostnames. It's owned by the platform team.
- **HTTPRoute**: the routing rules for one app. It's owned by the app team, in the app's namespace.

**In this project** Gateway API isn't installed yet:
```powershell
kubectl get gatewayclass
# real: error: the server doesn't have a resource type "gatewayclass"
```

**Try it.**
1. Install the Gateway API CRDs, using the `standard-install.yaml` of the latest release from https://github.com/kubernetes-sigs/gateway-api/releases:
   ```powershell
   kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/<version>/standard-install.yaml
   ```
2. Turn on Traefik's Gateway provider by adding this to [traefik-values.yaml](../traefik-values.yaml), then running `helm upgrade traefik traefik/traefik -n traefik -f k8s/traefik-values.yaml`:
   ```yaml
   providers:
     kubernetesGateway:
       enabled: true
   ```
   Check with `kubectl get gatewayclass` and `kubectl get gateway -A`. The Traefik chart creates a default GatewayClass and Gateway; note the Gateway's name and namespace.
3. Rewrite `demo-ingress` as an HTTPRoute. Save as `k8s/examples/httproute.yaml`:
   ```yaml
   apiVersion: gateway.networking.k8s.io/v1
   kind: HTTPRoute
   metadata:
     name: demo-route
   spec:
     parentRefs:
       - name: traefik-gateway          # the Gateway created by the chart (check its name)
         namespace: traefik
     hostnames: ["mylocal.localhost"]
     rules:
       - matches: [{path: {type: PathPrefix, value: /api}}]
         backendRefs: [{name: backend-service, port: 8080}]
       - matches: [{path: {type: PathPrefix, value: /}}]
         backendRefs: [{name: frontend-service, port: 80}]
   ```
   Compare it with [ingress.yaml](../ingress.yaml): same routes, without annotations. A canary is just two `backendRefs` with `weight: 90` and `weight: 10`.

New projects increasingly start with Gateway API, but Ingress is still everywhere, so it's worth knowing both.

---

### 15. StatefulSets done properly

**What it is.** A StatefulSet gives each pod a **stable identity**: a fixed name (`mongodb-0`, `mongodb-1`...), a fixed DNS name through a headless Service, and its **own PVC** that follows it. Pods start in order (0, then 1, then 2) and stop in reverse order.

**In this project:** MongoDB is a StatefulSet with **1 replica**, on purpose. Setting `replicas: 3` would **not** give you a replicated database: you'd get three separate, empty databases that know nothing about each other.

```powershell
kubectl get statefulset,pod,pvc -n demo-namespace -l app=mongodb
kubectl exec mongo-client -n demo-namespace -- getent hosts mongodb-0.mongodb
```
`mongodb-0.mongodb` is the stable per-pod DNS name. With more replicas you'd also get `mongodb-1.mongodb` and `mongodb-2.mongodb`.

**What a real 3-member MongoDB replica set needs:**
1. `replicas: 3`. Each pod gets its own PVC: `data-mongodb-0`, `-1`, `-2`.
2. Start `mongod` with `--replSet rs0`, and with a **keyfile** (`--keyFile`, stored in a Secret), so the members can authenticate to each other.
3. Run `rs.initiate()` **once**, listing the members by their stable names:
   ```js
   rs.initiate({ _id: "rs0", members: [
     { _id: 0, host: "mongodb-0.mongodb:27017" },
     { _id: 1, host: "mongodb-1.mongodb:27017" },
     { _id: 2, host: "mongodb-2.mongodb:27017" } ] })
   ```
4. Change the backend's connection to a replica-set URI:
   `mongodb://user:pass@mongodb-0.mongodb,mongodb-1.mongodb,mongodb-2.mongodb/belajar_db?replicaSet=rs0&authSource=admin`
5. Handle what happens after that: adding and removing members, failover, backups, upgrades one member at a time.

That last step is why teams use an **operator** (the MongoDB Community Operator or Percona Operator), which does all of this from one custom resource, or a managed service like Atlas ([production.md step 6](production.md#step-6-set-up-the-database-mongodb-atlas)). Doing it by hand once in kind is still the best way to understand StatefulSets.

**Other StatefulSet settings:**
- `podManagementPolicy: Parallel` starts all pods at once instead of in order.
- `updateStrategy.rollingUpdate.partition: 2` updates only pods with an index ≥ 2, which is a canary for stateful apps.
- `persistentVolumeClaimRetentionPolicy` controls whether PVCs are deleted when the StatefulSet is deleted or scaled down. The default keeps them, which is why your data survives `kubectl delete -k k8s/`.

---

### 16. Cluster lifecycle

**What it is.** Everything about keeping a cluster healthy over time: taking nodes out for maintenance, protecting apps while you do it, upgrading Kubernetes, and backing up the cluster's own state.

**Cordon and drain.**
- `cordon` marks a node unschedulable (no new pods).
- `drain` cordons it **and** evicts its pods so they move elsewhere.
- `uncordon` reverses both.

```powershell
kubectl cordon demo-cluster-worker
kubectl get nodes                       # worker shows Ready,SchedulingDisabled
kubectl drain demo-cluster-worker --ignore-daemonsets --delete-emptydir-data
kubectl get pods -n demo-namespace -o wide     # all app pods now on worker2
kubectl uncordon demo-cluster-worker
```
After uncordoning, the pods **don't move back** by themselves. Kubernetes doesn't rebalance running pods; a `kubectl rollout restart` spreads them again.

**PodDisruptionBudgets** protect apps during drains. A drain respects them and waits instead of evicting too many pods at once:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: backend
spec:
  minAvailable: 1
  selector: {matchLabels: {app: backend}}
```
With this applied, draining a node that holds both backend pods evicts one, waits until its replacement is Ready elsewhere, then evicts the other.

**Upgrading Kubernetes.**
- In a real cluster: upgrade the control plane one minor version at a time (1.36 → 1.37), then the add-ons, then the nodes one by one (drain, upgrade, uncordon). Managed services like EKS do the control-plane part for you.
- In kind there's no in-place upgrade. You recreate the cluster with a newer node image (`kind create cluster --image kindest/node:<version>`) and redeploy. That's quick because everything is in git: [README.md](../README.md#first-time-setup-run-from-the-repo-root).

**Backing up etcd.** etcd holds every object in the cluster. On self-managed clusters, take regular snapshots. In kind, etcd runs as a pod on the control plane, and its image includes `etcdctl` (run from PowerShell):
```powershell
kubectl exec -n kube-system etcd-demo-cluster-control-plane -- etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key snapshot save /var/lib/etcd/snapshot.db

kubectl exec -n kube-system etcd-demo-cluster-control-plane -- etcdctl --write-out=table snapshot status /var/lib/etcd/snapshot.db
```
An etcd snapshot doesn't contain the data in volumes (MongoDB's files); those need their own backups ([topic 4](#4-jobs-and-cronjobs)). Managed services back up etcd for you, so there you focus on app data and on keeping your manifests in git.

---

## Part 4: Concepts worth understanding

These are ideas rather than objects, but they explain why Kubernetes behaves as it does.

### Declarative vs. imperative

- **Imperative:** "do this now". Examples: `kubectl run`, `kubectl create`, `kubectl scale`, `kubectl set image`, `kubectl delete`.
- **Declarative:** "this is how it should be". `kubectl apply -k k8s/` with files kept in git.

Imperative commands are great for learning and debugging; most **Try it** blocks above use them. But the next `kubectl apply -k k8s/` overwrites them with what the files say. For anything you want to keep, change the files.

```powershell
kubectl scale deploy/frontend --replicas=3 -n demo-namespace   # imperative
kubectl apply -k k8s/                                            # declarative: back to 2
kubectl diff -k k8s/                                             # preview what apply would change
```

### The reconciliation loop

Every controller runs the same loop: **observe** the actual state, **compare** it with the desired state, **act** to close the gap, and repeat forever. That's why a deleted pod comes back, why a ReplicaSet always has the right count, and why Argo CD's self-heal works.

```powershell
kubectl delete pod -n demo-namespace -l app=frontend --wait=false
kubectl get pods -n demo-namespace -l app=frontend -w     # replacements appear within seconds
```
You never told Kubernetes to "restart" anything. The ReplicaSet saw 0 pods where 2 should be and created new ones.

### Everything is an API object, and CRDs extend the API

`kubectl` is just a client for a REST API. Every object (Pod, Service, Ingress) is a resource type in that API:
```powershell
kubectl api-resources --namespaced=true      # every namespaced type the cluster knows (real: 58)
kubectl explain deployment.spec.strategy     # built-in documentation for any field
kubectl get --raw /apis/apps/v1 | Select-Object -First 1
```
`kubectl explain` is worth using often: it documents every field of every object.

**Custom Resource Definitions (CRDs)** add new types. Installing Traefik added 25 of them to your cluster:
```powershell
kubectl get crd
```
*Real (trimmed):*
```
NAME                         SCOPE        CREATED AT
ingressroutes.traefik.io     Namespaced   2026-11-17T09:01:55Z
middlewares.traefik.io       Namespaced   2026-11-17T09:01:55Z
tlsoptions.traefik.io        Namespaced   2026-11-17T09:01:55Z
...
```
An **operator** is a CRD plus a controller that reconciles it (topic 15's MongoDB operator, topic 11's `ServiceMonitor`, topic 13's `Application`). It's the same reconciliation loop, applied to your own types.

### The networking model

Kubernetes networking rests on a few rules:
1. Every pod gets its **own IP**, and every pod can reach every other pod's IP without NAT. On your cluster: `10.244.1.x` on worker, `10.244.2.x` on worker2, routed by kindnet.
2. Pod IPs change whenever pods are recreated. So clients use a **Service**: a stable virtual IP (`backend-service` → `10.96.44.91`) that kube-proxy load-balances across Ready pods.
3. **DNS** (CoreDNS) gives Services names: `backend-service.demo-namespace.svc.cluster.local`.
4. Traffic from outside comes in through a **NodePort**, a **LoadBalancer** Service or an **Ingress/Gateway** controller (Traefik here).

The DNS and endpoints sections of [troubleshooting.md](troubleshooting.md#failure-9-dns-doesnt-resolve) show these rules on your cluster.

**Service types**, side by side:

| Type | Reachable from | In this project |
|---|---|---|
| `ClusterIP` | Inside the cluster only | `backend-service`, `frontend-service` |
| Headless (`clusterIP: None`) | Inside; DNS returns pod IPs directly | `mongodb` |
| `NodePort` | Every node's IP on a port 30000–32767 | Not used |
| `LoadBalancer` | An external load balancer (cloud) | Not used in kind; the ALB on EKS is similar |
| `ExternalName` | A DNS alias to something outside | Not used |

### The 12-factor app

A set of rules for apps that run well on platforms like Kubernetes. Your backend already follows most of them:

| Factor | Meaning | In this project |
|---|---|---|
| Config in the environment | No settings baked into the image | `SPRING_DATA_MONGODB_*` from the ConfigMap and Secret |
| Stateless processes | Any instance can serve any request | 2 interchangeable backend pods; state lives in MongoDB |
| Logs to stdout | The platform collects them | `kubectl logs` works with no log files |
| Disposability | Fast start, clean shutdown | Probes and rolling updates depend on it |
| Dev/prod parity | Same image everywhere | One image, different config for kind and EKS |
| Backing services as resources | The database is just a URL | Swap MongoDB for Atlas by changing config ([production.md](production.md)) |

---

## Suggested order for this project

| # | Topic | Why this order | Effort |
|---|---|---|---|
| 1 | [metrics-server + HPA](#7-autoscaling-hpa-and-metrics-server) | Quick and very visible; teaches requests too | 30 min |
| 2 | [CronJob backup of MongoDB](#4-jobs-and-cronjobs) | Practical, and covers Jobs, PVCs and Secrets together | 30 min |
| 3 | [Init container waiting for MongoDB](#5-init-containers-and-sidecars) | Addresses a real startup error in your logs | 15 min |
| 4 | [Rolling update to 1.0.1 and rollback](#2-rolling-updates-in-depth) | The everyday deployment workflow | 20 min |
| 5 | [RBAC read-only viewer](#9-rbac-and-serviceaccounts) | Security basics, fully testable with `auth can-i` | 20 min |
| 6 | [Prometheus and Grafana](#11-observability-prometheus-and-grafana) | Shows your app from the inside | 1 hour |
| 7 | [Argo CD on your GitHub repo](#13-gitops-with-argo-cd) | How teams actually deploy | 1 hour |
| 8 | [NetworkPolicy on a Calico cluster](#8-networkpolicy) | Needs a second cluster, so do it later | 1 hour |
| 9 | [MongoDB replica set](#15-statefulsets-done-properly) | The hardest one; shows why operators exist | Half a day |

The effort column is a rough guess for someone at your stage, not a measurement.

**Official references:**
- [Kubernetes documentation: Concepts](https://kubernetes.io/docs/concepts/)
- [Tasks](https://kubernetes.io/docs/tasks/)
- [kubectl reference](https://kubernetes.io/docs/reference/kubectl/)
- the interactive [Kubernetes Basics tutorial](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
