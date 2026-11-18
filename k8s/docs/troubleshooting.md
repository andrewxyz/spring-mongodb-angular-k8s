# Troubleshooting and debugging in Kubernetes

How to find out why something in this project's cluster isn't working, and how to fix it. The first part is a general method and the tools; the second part goes through the most common failures one by one, using `kind-demo-cluster` and `demo-namespace` as the example. Outputs marked *real* were taken from this cluster when it was healthy, so you can compare.

For how the cluster is built (control plane, workers, system pods), see [cluster-architecture.md](cluster-architecture.md).

> The examples use PowerShell. In Git Bash, prefix `docker exec` and `kubectl exec` commands that contain `/paths` with `MSYS_NO_PATHCONV=1`.

## Part 1: The method

### Work from the outside in

```
1. kubectl get        → what is the STATUS?            (the symptom)
2. kubectl describe   → what do the Events say?        (Kubernetes' point of view)
3. kubectl logs       → what does the app say?         (the app's point of view)
4. kubectl exec       → test from inside the cluster   (DNS, network, files)
5. kubectl port-forward → test one pod or Service directly, skipping the Ingress
6. fix → apply → watch:  kubectl get pods -n demo-namespace -w
```

Most problems are solved by step 2 or 3. Read the **Events** at the bottom of `describe`, and the **last lines** of the logs.

### Start with the status

```powershell
kubectl get pods -n demo-namespace -o wide
```
*Real (healthy):*
```
NAME                        READY   STATUS    RESTARTS   AGE   IP           NODE
backend-6458c7bc99-rspb8    1/1     Running   0          19h   10.244.1.2   demo-cluster-worker
backend-6458c7bc99-xwd4p    1/1     Running   0          19h   10.244.2.8   demo-cluster-worker2
frontend-594fdb5667-khp2d   1/1     Running   0          19h   10.244.1.3   demo-cluster-worker
frontend-594fdb5667-lc5z8   1/1     Running   0          19h   10.244.2.3   demo-cluster-worker2
mongo-client                1/1     Running   0          19h   10.244.2.4   demo-cluster-worker2
mongodb-0                   1/1     Running   0          19h   10.244.2.7   demo-cluster-worker2
```

The STATUS column tells you where to look next:

| STATUS / READY | Meaning | Go to |
|---|---|---|
| `Pending` | No node has been picked, or a volume isn't ready | [Failure 3](#failure-3-pod-stuck-in-pending) |
| `ErrImagePull`, `ImagePullBackOff` | The node can't get the image | [Failure 2](#failure-2-errimagepull--imagepullbackoff) |
| `CreateContainerConfigError` | A Secret or ConfigMap the pod needs is missing | [Failure 4](#failure-4-createcontainerconfigerror) |
| `CrashLoopBackOff`, `Error` | The container starts, then exits | [Failure 5](#failure-5-crashloopbackoff) |
| `OOMKilled` (in describe) | The container used more memory than its limit | [Failure 6](#failure-6-oomkilled) |
| `Running` but `0/1` | Readiness probe is failing | [Failure 7](#failure-7-running-but-not-ready-01) |
| `Running`, `1/1`, but the app is broken | Look at Services, DNS, the Ingress or the database | [Failures 8–12](#failure-8-service-has-no-endpoints) |
| RESTARTS keeps going up | The container keeps dying (crash, OOM or liveness probe) | [Failures 5, 6, 7](#failure-5-crashloopbackoff) |

### The toolbox

```powershell
# What is there, and its state
kubectl get all -n demo-namespace
kubectl get pods -n demo-namespace -o wide --show-labels

# Why: details and Events of one object
kubectl describe pod <pod> -n demo-namespace
kubectl describe deploy backend -n demo-namespace

# All recent events in the namespace, newest last
kubectl get events -n demo-namespace --sort-by=.lastTimestamp

# Logs
kubectl logs <pod> -n demo-namespace                  # current container
kubectl logs <pod> -n demo-namespace --previous       # the container that crashed before this one
kubectl logs deploy/backend -n demo-namespace -f      # follow one pod of the Deployment
kubectl logs -l app=backend -n demo-namespace --tail=50 --prefix   # all backend pods

# The full object as Kubernetes sees it (with status)
kubectl get pod <pod> -n demo-namespace -o yaml

# Inside the cluster
kubectl exec -it mongo-client -n demo-namespace -- sh
kubectl exec -it deploy/frontend -n demo-namespace -- sh

# Straight to a Service or pod, skipping Traefik
kubectl port-forward svc/backend-service 8080:8080 -n demo-namespace
#   then open http://localhost:8080/api/tutorials or /actuator/health

# A temporary debug container next to an app container (useful when the image has no shell or tools)
kubectl debug -it <backend-pod> -n demo-namespace --image=busybox:1.37 --target=backend

# Deployment history and rollback
kubectl rollout status  deploy/backend -n demo-namespace
kubectl rollout history deploy/backend -n demo-namespace
kubectl rollout undo    deploy/backend -n demo-namespace
```

`mongo-client` ([debug/mongo-client-pod.yaml](../debug/mongo-client-pod.yaml)) is handy for network tests because it has a shell and `getent`. See [mongodb-client.md](mongodb-client.md) for using it as a database client.

## Part 2: Common failures

Each failure has the same layout: **symptom**, **check**, **common causes in this project**, **fix**. Some have a **Try it** block that breaks something on purpose so you can practise. Those change the cluster, and each one includes the undo command.

---

### Failure 1: "No resources found"

**Symptom**
```powershell
kubectl get pods
# No resources found in default namespace.
```

**Check**
```powershell
kubectl config current-context        # expect kind-demo-cluster
kubectl get pods -A                   # every namespace
```

**Causes**
- `kubectl` looks in the `default` namespace unless told otherwise. This project's pods are in `demo-namespace`.
- `kubectl` points at another cluster (another kind cluster, Docker Desktop's built-in Kubernetes, a cloud cluster).

**Fix**
```powershell
kubectl get pods -n demo-namespace
kubectl config set-context --current --namespace=demo-namespace   # make it the default
kubectl config use-context kind-demo-cluster                      # if the context was wrong
```

---

### Failure 2: ErrImagePull / ImagePullBackOff

**Symptom:** the pod shows `ErrImagePull`, then `ImagePullBackOff` (Kubernetes retries with growing delays).

**Check**
```powershell
kubectl describe pod <pod> -n demo-namespace     # Events: "Failed to pull image ... not found"
kubectl get deploy backend -n demo-namespace -o jsonpath='{.spec.template.spec.containers[0].image}'
# real: belajar-backend:1.0.0

docker exec demo-cluster-worker crictl images     # which images the node actually has
```
*Real (healthy):*
```
IMAGE                                TAG     IMAGE ID        SIZE
docker.io/library/belajar-backend    1.0.0   ea5a390388e7b   145MB
docker.io/library/belajar-frontend   1.0.0   2c724d8d3e78f   26.1MB
```

**Causes in this project**
- The image was built but **not loaded into kind**. kind nodes can't see the images in Docker Desktop; they need `kind load`.
- The tag in the Deployment doesn't match the built tag (a typo, `1.0.1` vs `1.0.0`).
- `imagePullPolicy: Always` on a local-only image: the node tries Docker Hub, which doesn't have it.
- For a private registry (Docker Hub private repo, ECR): no `imagePullSecrets`. See [dockerhub.md](dockerhub.md).

**Fix**
```powershell
docker build -t belajar-backend:1.0.0 spring-boot-data-mongodb
kind load docker-image belajar-backend:1.0.0 --name demo-cluster
kubectl rollout restart deploy/backend -n demo-namespace
```

**Try it**
```powershell
kubectl set image deploy/frontend frontend=belajar-frontend:9.9.9 -n demo-namespace
kubectl get pods -n demo-namespace -l app=frontend -w
```
One new pod goes to `ImagePullBackOff`, but the **two old pods keep running**, so the site stays up. A rolling update only removes old pods once new ones are Ready. Undo:
```powershell
kubectl rollout undo deploy/frontend -n demo-namespace
```

---

### Failure 3: Pod stuck in Pending

**Symptom:** STATUS `Pending` for more than a few seconds, and NODE is `<none>`.

**Check**
```powershell
kubectl describe pod <pod> -n demo-namespace
# Events: Warning  FailedScheduling  0/3 nodes are available: ...
kubectl describe node demo-cluster-worker | Select-String -Context 0,8 "Allocated resources"
kubectl get pvc -n demo-namespace
```

The `FailedScheduling` message says exactly why. Common ones:

| Message contains | Cause | Fix |
|---|---|---|
| `Insufficient cpu` / `Insufficient memory` | The pod's **requests** don't fit on any node | Lower the requests, or add nodes |
| `untolerated taint {node-role.kubernetes.io/control-plane: }` | Only the control plane would fit, and the pod doesn't tolerate its taint | Normal for app pods; free up the workers |
| `didn't match Pod's node affinity/selector` | A `nodeSelector` label isn't on any node | Fix the label or the selector (Traefik uses `ingress-ready=true`) |
| `pod has unbound immediate PersistentVolumeClaims` | The PVC isn't Bound | See below |
| `node(s) had volume node affinity conflict` | The PVC's disk lives on a node the pod can't use | See [Failure 12](#failure-12-mongodb-data-is-gone-or-the-pvc-is-stuck) |

**PVCs:**
```powershell
kubectl get pvc -n demo-namespace
kubectl get storageclass
```
*Real (healthy):*
```
NAME             STATUS   VOLUME                                     CAPACITY   STORAGECLASS
data-mongodb-0   Bound    pvc-0891ab09-bdf6-4248-a6fa-f0c54bfb7fa8   1Gi        standard

NAME                 PROVISIONER             VOLUMEBINDINGMODE
standard (default)   rancher.io/local-path   WaitForFirstConsumer
```
`WaitForFirstConsumer` means a new PVC stays `Pending` **until a pod uses it**. That's normal, and not an error by itself. A PVC that stays `Pending` while its pod is also `Pending` usually means a wrong `storageClassName`.

---

### Failure 4: CreateContainerConfigError

**Symptom:** STATUS `CreateContainerConfigError`. The image is fine, but the container can't be created.

**Check**
```powershell
kubectl describe pod <pod> -n demo-namespace
# Events: Error: secret "mongodb-secret" not found
#     or: Error: couldn't find key MONGO_INITDB_ROOT_PASSWORD in Secret demo-namespace/mongodb-secret

kubectl get secret,configmap -n demo-namespace
kubectl get secret mongodb-secret -n demo-namespace -o jsonpath='{.data}'   # key names (values are base64)
```

**Causes in this project**
- `mongodb-secret` or `backend-config` wasn't applied, or was applied to another namespace. Running `kubectl apply -f` on single files without `-n demo-namespace` puts them in `default`.
- A key was renamed in the Secret but not in the Deployment's `secretKeyRef`.

**Fix:** apply everything together, so the namespace is always right:
```powershell
kubectl apply -k k8s/
```

---

### Failure 5: CrashLoopBackOff

**Symptom:** STATUS `CrashLoopBackOff` or `Error`, and RESTARTS keeps increasing. The container starts, exits, and Kubernetes waits longer before each restart.

**Check:** the logs of the **previous** (crashed) container are the important ones:
```powershell
kubectl logs <pod> -n demo-namespace --previous
kubectl describe pod <pod> -n demo-namespace     # "Last State: Terminated, Reason: Error, Exit Code: 1"
```

**Causes in this project**

| Log shows | Cause | Fix |
|---|---|---|
| `MongoSecurityException: Exception authenticating` | Wrong MongoDB username or password | See the password gotcha below |
| `APPLICATION FAILED TO START` + a Spring message | Bad configuration or env var | Read the message; fix `backend-config` or the code |
| `Unresolved compilation problem` | The image was built from broken classes | Rebuild from clean: `./mvnw clean package` |
| nginx `emerg` ... `default.conf` | A syntax error in the nginx ConfigMap | Fix [frontend/configmap.yaml](../frontend/configmap.yaml), apply, restart |

**The MongoDB password gotcha.** The mongo image reads `MONGO_INITDB_ROOT_USERNAME` and `MONGO_INITDB_ROOT_PASSWORD` **only the first time**, when its data folder is empty. If you change the password in [mongodb/secret.yaml](../mongodb/secret.yaml) after the PVC exists, the backend gets the new password but the database still has the old one, so authentication fails. Fix it either way:
- put the old password back in the Secret, or
- change the password inside MongoDB (with `mongosh`, `db.changeUserPassword(...)`), or
- in this learning setup only, wipe the data: delete the StatefulSet and the PVC `data-mongodb-0`, then apply again.

---

### Failure 6: OOMKilled

**Symptom:** the pod restarts now and then; `describe` shows the reason.

**Check**
```powershell
kubectl describe pod <pod> -n demo-namespace
#   Last State:  Terminated
#     Reason:    OOMKilled
#     Exit Code: 137
```

Exit code **137** means the container was killed. Together with `OOMKilled`, it means the container used more memory than its `limits.memory`.

**In this project:** the backend's limit is `768Mi` ([backend/deployment.yaml](../backend/deployment.yaml)). The JVM is started with `-XX:MaxRAMPercentage=75`, so its heap stays under about 576Mi, and the rest is for the JVM's own memory. The frontend's limit is `128Mi`, which is plenty for nginx serving static files.

**Fix:** raise `limits.memory` (and `requests.memory`), apply, and watch. If memory keeps growing over time, it's a leak in the app, and a bigger limit only delays the crash.

---

### Failure 7: Running but not Ready (0/1)

**Symptom:** STATUS `Running` but READY `0/1`. The pod gets no traffic, because only Ready pods are added to a Service.

**Check**
```powershell
kubectl describe pod <pod> -n demo-namespace
# Events: Warning  Unhealthy  Readiness probe failed: HTTP probe failed with statuscode: 503
#     or: Startup probe failed: Get "http://10.244.1.2:8080/actuator/health/liveness": connection refused
```

Test the same URL yourself:
```powershell
kubectl port-forward <backend-pod> 8080:8080 -n demo-namespace
# in another terminal:
curl.exe http://localhost:8080/actuator/health/readiness
```

**Causes in this project**
- The backend is still starting. The JVM takes a while, which is why it has a `startupProbe` allowing up to about 2.5 minutes. `connection refused` in the first minute is normal.
- The probe path or port is wrong after an edit.
- A `livenessProbe` that fails makes Kubernetes **restart** the container. A failing `readinessProbe` only takes it out of the Service.

**"Ready" doesn't mean "working".** The backend's readiness check doesn't include MongoDB. If MongoDB is down, the backend pods stay `1/1 Ready`, but API calls fail.

**Try it**
```powershell
kubectl scale statefulset mongodb --replicas=0 -n demo-namespace
kubectl get pods -n demo-namespace            # the backends stay 1/1
curl.exe http://mylocal.localhost/api/tutorials   # fails after a timeout (500)
kubectl logs deploy/backend -n demo-namespace --tail=20
kubectl scale statefulset mongodb --replicas=1 -n demo-namespace    # undo
```

---

### Failure 8: Service has no endpoints

**Symptom:** pods are Running and Ready, but calling the Service fails (`connection refused`, or 503 from Traefik).

**Check:** a Service sends traffic to its **endpoints**, the IPs of the Ready pods that match its selector:
```powershell
kubectl get endpointslices -n demo-namespace
```
*Real (healthy):*
```
NAME                     ADDRESSTYPE   PORTS   ENDPOINTS
backend-service-tzrjw    IPv4          8080    10.244.1.2,10.244.2.8
frontend-service-64v55   IPv4          80      10.244.1.3,10.244.2.3
mongodb-6jbdx            IPv4          27017   10.244.2.7
```

Compare them with the pod IPs from `kubectl get pods -o wide`. If ENDPOINTS is `<unset>` or empty:
```powershell
kubectl get svc backend-service -n demo-namespace -o jsonpath='{.spec.selector}'   # {"app":"backend"}
kubectl get pods -n demo-namespace --show-labels                                   # do pods have app=backend?
```

**Causes**
- The Service's `selector` doesn't match the pod labels (a typo, or a label changed in only one file).
- `targetPort` is wrong. Here it's the named port `http`, so the container must have a port named `http`.
- The pods aren't Ready ([Failure 7](#failure-7-running-but-not-ready-01)).

**Fix:** make the selector and labels match, and apply.

---

### Failure 9: DNS doesn't resolve

**Symptom:** logs say `UnknownHostException`, `Temporary failure in name resolution` or `could not resolve host`.

**Check from inside the cluster:**
```powershell
kubectl exec mongo-client -n demo-namespace -- getent hosts backend-service mongodb frontend-service
kubectl exec mongo-client -n demo-namespace -- cat /etc/resolv.conf
kubectl get pods -n kube-system -l k8s-app=kube-dns
```
*Real (healthy):*
```
10.96.44.91     backend-service.demo-namespace.svc.cluster.local
10.244.2.7      mongodb.demo-namespace.svc.cluster.local
10.96.225.88    frontend-service.demo-namespace.svc.cluster.local

search demo-namespace.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
```

How to read it:
- `backend-service` resolves to a **Service IP** (`10.96.x.x`), which kube-proxy load-balances across the pods.
- `mongodb` is a **headless** Service (`clusterIP: None`), so it resolves straight to the **pod IP** (`10.244.2.7`). It only resolves while `mongodb-0` is Ready.
- The `search` line lets a short name like `mongodb` work inside the same namespace. From **another namespace**, use `mongodb.demo-namespace` or the full `mongodb.demo-namespace.svc.cluster.local`.

**A real example from this cluster.** The backend's log from when the cluster was created contains:
```
09:05:29  Exception in monitor thread while connecting to server mongodb:27017
          com.mongodb.MongoSocketException: mongodb: Temporary failure in name resolution
09:07:50  Monitor thread successfully connected to server ... address=mongodb:27017
```
The backend started before `mongodb-0` was Ready, so the headless name didn't exist yet. The MongoDB driver kept retrying and connected once MongoDB was up. This is normal at startup and needs no fix. It only becomes a problem if the error never stops.

**Causes when it persists**
- A wrong Service name, or a missing namespace in the name.
- The target pods are not Ready (for headless Services).
- CoreDNS pods aren't running (`kubectl get pods -n kube-system -l k8s-app=kube-dns`).

---

### Failure 10: The browser gets 404, 502/503 or "can't connect"

The path of a request is: browser → port 80 on your PC → kind control-plane container → Traefik → Service → pod. Test each hop.

**Check**
```powershell
curl.exe -s -o NUL -w "%{http_code}`n" http://mylocal.localhost/              # real: 200
curl.exe -s -o NUL -w "%{http_code}`n" http://mylocal.localhost/api/tutorials # real: 200
curl.exe -s -o NUL -w "%{http_code}`n" -H "Host: other.localhost" http://127.0.0.1/   # real: 404

kubectl get ingress -n demo-namespace
kubectl describe ingress demo-ingress -n demo-namespace
kubectl get pods -n traefik -o wide
kubectl logs -n traefik deploy/traefik --tail=50
docker ps --filter name=demo-cluster-control-plane --format "{{.Ports}}"   # 0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp
```
`curl.exe` is used because in Windows PowerShell 5.1 plain `curl` is an alias for `Invoke-WebRequest`.

*Real (healthy):*
```
NAME           CLASS     HOSTS               ADDRESS   PORTS
demo-ingress   traefik   mylocal.localhost             80
```
An empty ADDRESS is normal here: Traefik's Service is ClusterIP and the traffic arrives through a `hostPort`, so there's no load-balancer address to show.

| Result | Meaning | Look at |
|---|---|---|
| Can't connect at all | Nothing listens on port 80 | Is Docker running? Is the Traefik pod Running? Are ports 80/443 in `docker ps`? Is another program (IIS, Skype, another web server) using port 80? |
| `404 page not found` (Traefik's plain page) | No Ingress rule matched | The host (only `mylocal.localhost` matches), `ingressClassName: traefik`, the path |
| `503` | The rule matched, but the Service has no Ready endpoints | [Failure 7](#failure-7-running-but-not-ready-01), [Failure 8](#failure-8-service-has-no-endpoints) |
| `502` | Traefik reached a pod but it didn't answer properly | Wrong Service port, or the app crashed mid-request; check the app logs |
| Page loads but the list is empty and API calls fail | Frontend is fine; the backend or MongoDB isn't | `curl.exe http://mylocal.localhost/api/tutorials`, then the backend logs |
| `500` from `/api/...` | The backend got the request and threw an error | `kubectl logs -l app=backend -n demo-namespace --tail=100` |

Skip the Ingress entirely to tell Traefik problems apart from app problems:
```powershell
kubectl port-forward svc/frontend-service 8081:80 -n demo-namespace     # http://localhost:8081
kubectl port-forward svc/backend-service  8080:8080 -n demo-namespace   # http://localhost:8080/api/tutorials
```
If these work, the problem is in the Ingress or Traefik. If they don't, it's in the app.

---

### Failure 11: My code change doesn't show up

**Symptom:** you rebuilt the image and ran `kubectl apply`, but the app behaves the same.

**Check**
```powershell
kubectl get pods -n demo-namespace -l app=backend     # AGE: were the pods actually replaced?
kubectl describe pod <pod> -n demo-namespace | Select-String "Image ID"
docker exec demo-cluster-worker crictl images | Select-String belajar
```

**Causes**
- `kubectl apply` of an **unchanged** manifest does nothing. Same tag means same spec, so no rollout happens.
- The new image was built but not `kind load`-ed, so the nodes still have the old one under the same tag.
- `imagePullPolicy: IfNotPresent` plus the same tag means a node that already has `1.0.0` keeps using it.

**Fix:** load the new image and force a restart:
```powershell
kind load docker-image belajar-backend:1.0.0 --name demo-cluster
kubectl rollout restart deploy/backend -n demo-namespace
```
The cleaner habit is a **new tag for every change** (`1.0.1`, `1.0.2`), updated in the Deployment. See [After changing app code](../README.md#after-changing-app-code).

---

### Failure 12: MongoDB data is gone, or the PVC is stuck

**Check**
```powershell
kubectl get statefulset,pod,pvc -n demo-namespace -l app=mongodb
kubectl get pvc data-mongodb-0 -n demo-namespace
kubectl get pv
kubectl describe pod mongodb-0 -n demo-namespace
```

**Facts that explain most surprises**
- Deleting the **pod** `mongodb-0` is safe: the StatefulSet recreates it with the same PVC, and the data is kept.
- `kubectl delete -k k8s/` deletes the StatefulSet but **not** the PVC created from `volumeClaimTemplates`. Your data survives and comes back on the next apply.
- Deleting the **PVC** deletes the data: the `standard` StorageClass has reclaim policy `Delete`.
- Deleting the **kind cluster** deletes everything, data included.
- local-path volumes live on one node (here worker2). If that node is gone, `mongodb-0` stays `Pending` with `volume node affinity conflict`. In this learning setup the fix is to delete the PVC (losing the data) and let a new one be created.

---

### Failure 13: A node is NotReady

**Symptom:**
```powershell
kubectl get nodes
# demo-cluster-worker2   NotReady   <none>   ...
```
Pods on that node go `Terminating` or `Unknown` after a few minutes and are recreated elsewhere (except `mongodb-0`, whose disk is on that node).

**Check**
```powershell
kubectl describe node demo-cluster-worker2          # Conditions: Ready=Unknown "Kubelet stopped posting node status"
docker ps -a --filter name=demo-cluster             # is the node container running?
docker exec demo-cluster-worker2 systemctl status kubelet containerd --no-pager
docker exec demo-cluster-worker2 journalctl -u kubelet --no-pager -n 50
```

**Causes in kind**
- The node container stopped (Docker Desktop restarted, the PC slept, low memory).
- The kubelet or containerd inside the node crashed.
- Docker Desktop is low on memory or disk (look for `MemoryPressure` / `DiskPressure` = True).

**Fix**
```powershell
docker start demo-cluster-worker2                   # if the container is stopped
docker exec demo-cluster-worker2 systemctl restart kubelet
```
If the cluster stays broken, recreating it is quick and is often the best fix for a learning cluster. The full steps are in [README.md](../README.md#first-time-setup-run-from-the-repo-root).

---

### Failure 14: kubectl can't reach the cluster

**Symptom**
```
Unable to connect to the server: dial tcp 127.0.0.1:62526: connectex: No connection could be made...
```

**Check and fix**
```powershell
docker ps --filter name=demo-cluster                # are the 3 node containers Up?
kind get clusters                                   # does demo-cluster still exist?
kubectl config current-context
```
- Docker Desktop isn't running: start it and wait a minute; the kind containers start with it.
- The containers are stopped: `docker start demo-cluster-control-plane demo-cluster-worker demo-cluster-worker2`.
- The kubeconfig entry is stale (for example after recreating the cluster): `kind export kubeconfig --name demo-cluster`.

---

### Failure 15: A bad deploy, and how to roll back

**Symptom:** after an update, `kubectl rollout status` never finishes, or the new version misbehaves.

**Check**
```powershell
kubectl rollout status  deploy/backend -n demo-namespace     # waits; times out on a stuck rollout
kubectl get rs -n demo-namespace -l app=backend              # old and new ReplicaSets side by side
kubectl rollout history deploy/backend -n demo-namespace
```

**Fix**
```powershell
kubectl rollout undo deploy/backend -n demo-namespace        # back to the previous version
kubectl rollout undo deploy/backend -n demo-namespace --to-revision=2
```
A rollback with `kubectl` is temporary: the next `kubectl apply -k k8s/` brings back whatever the files say. Fix the files too.

## Quick reference

| Symptom | First command | Usual cause here |
|---|---|---|
| No resources found | `kubectl get pods -A` | Missing `-n demo-namespace` |
| ImagePullBackOff | `kubectl describe pod` | Forgot `kind load`, or a tag typo |
| Pending | `kubectl describe pod` (FailedScheduling) | Requests too big, selector, PVC |
| CreateContainerConfigError | `kubectl describe pod` | Secret/ConfigMap missing or in the wrong namespace |
| CrashLoopBackOff | `kubectl logs --previous` | Wrong Mongo password, bad config |
| OOMKilled / exit 137 | `kubectl describe pod` (Last State) | Memory limit too low |
| 0/1 Ready | `kubectl describe pod` (probe events) | Still starting, or wrong probe |
| Service doesn't answer | `kubectl get endpointslices` | Selector doesn't match labels |
| UnknownHost / name resolution | `kubectl exec mongo-client -- getent hosts <name>` | Wrong name, or target not Ready |
| Browser 404 | `kubectl describe ingress` | Wrong host or ingress class |
| Browser 503 | `kubectl get endpointslices` | No Ready pods behind the Service |
| Change doesn't show | `kubectl get pods` (AGE) | Same tag, no restart |
| Node NotReady | `kubectl describe node`, `docker ps -a` | Node container stopped |
| Can't connect to server | `docker ps` | Docker Desktop not running |
