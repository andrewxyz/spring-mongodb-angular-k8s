# Inspecting the cluster: control plane and worker nodes

How to look at the Kubernetes architecture of this project's `kind-demo-cluster` with `kubectl`: which node is the control plane (master), which are workers, what runs where, and how to check that each part is healthy. The outputs below are real, taken from the cluster (Kubernetes v1.37.0) and trimmed.

Every command here is read-only, except the experiments in section 10. For fixing problems, see [troubleshooting.md](troubleshooting.md).

## The big picture

```
                  ┌──────────── demo-cluster-control-plane (172.18.0.8) ─────────────┐
 kubectl ───────► │ kube-apiserver   etcd   kube-scheduler   kube-controller-manager │  "the brain"
 (127.0.0.1:62526)│ + kubelet, kube-proxy, kindnet, coredns, traefik                  │
                  └───────────────────────────────┬──────────────────────────────────┘
                           ┌──────────────────────┴──────────────────────┐
      demo-cluster-worker (172.18.0.5)                 demo-cluster-worker2 (172.18.0.7)
      kubelet, kube-proxy, kindnet, containerd         kubelet, kube-proxy, kindnet, containerd
      backend, frontend                                backend, frontend, mongodb-0, mongo-client
```

- The **control plane** decides what should run and where, and stores the cluster's state.
- The **workers** run your app pods.
- In kind, each "node" is a Docker container (`kindest/node:v1.37.0`). On EKS or GKE they would be virtual machines.

## 1. Which cluster am I talking to?

```powershell
kubectl config current-context
# kind-demo-cluster

kubectl cluster-info
# Kubernetes control plane is running at https://127.0.0.1:62526
# CoreDNS is running at https://127.0.0.1:62526/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

Every `kubectl` command is an HTTPS call to the **API server**. kind forwards a port on your PC (here 62526) to the control-plane container.

## 2. List the nodes and their roles

```powershell
kubectl get nodes -o wide
```
```
NAME                         STATUS   ROLES           VERSION   INTERNAL-IP   OS-IMAGE                       CONTAINER-RUNTIME
demo-cluster-control-plane   Ready    control-plane   v1.37.0   172.18.0.8    Debian GNU/Linux 13 (trixie)   containerd://2.3.4
demo-cluster-worker          Ready    <none>          v1.37.0   172.18.0.5    Debian GNU/Linux 13 (trixie)   containerd://2.3.4
demo-cluster-worker2         Ready    <none>          v1.37.0   172.18.0.7    Debian GNU/Linux 13 (trixie)   containerd://2.3.4
```

- **ROLES** `control-plane` marks the master node. Workers show `<none>`, which is normal.
- **STATUS** `Ready` means the node's kubelet is healthy and reporting in.
- **CONTAINER-RUNTIME** `containerd` is the program that actually runs the containers on each node.

## 3. How Kubernetes knows a node's role: labels

```powershell
kubectl get nodes --show-labels
```

The control plane has these labels:
```
node-role.kubernetes.io/control-plane=
ingress-ready=true                                   # added by k8s/kind-config.yaml
node.kubernetes.io/exclude-from-external-load-balancers=
```

The workers only have the basic ones (`kubernetes.io/hostname`, `kubernetes.io/os`, `kubernetes.io/arch`). The ROLES column is built from labels that start with `node-role.kubernetes.io/`.

To show only the control-plane nodes, or only the workers:

```powershell
kubectl get nodes -l node-role.kubernetes.io/control-plane     # control plane
kubectl get nodes -l '!node-role.kubernetes.io/control-plane'  # workers
```

## 4. Why app pods never land on the control plane: taints

```powershell
kubectl describe node demo-cluster-control-plane | Select-String Taints
# Taints:             node-role.kubernetes.io/control-plane:NoSchedule
```

A **taint** works like a "keep out" sign. `NoSchedule` tells the scheduler not to place pods on that node unless the pod **tolerates** the taint. The backend, frontend and mongodb pods have no toleration, so they only run on workers.

Traefik does run on the control plane. That's deliberate: [traefik-values.yaml](../traefik-values.yaml) gives it a toleration and a `nodeSelector: ingress-ready=true`, because that's the node where ports 80 and 443 are mapped to your PC.

## 5. The control-plane components

```powershell
kubectl get pods -n kube-system -o wide
```
```
NAME                                                 NODE
etcd-demo-cluster-control-plane                      demo-cluster-control-plane
kube-apiserver-demo-cluster-control-plane            demo-cluster-control-plane
kube-controller-manager-demo-cluster-control-plane   demo-cluster-control-plane
kube-scheduler-demo-cluster-control-plane            demo-cluster-control-plane
coredns-559f6c778d-9gjcb / -qmdpp                    demo-cluster-control-plane
kube-proxy-6kkr8 / -bk2jc / -ntttl                   one per node
kindnet-64h8j / -g8pdj / -q8b29                      one per node
```

| Component | Where | What it does |
|---|---|---|
| **kube-apiserver** | control plane | The front door. `kubectl`, the kubelets and the controllers all talk only to it. |
| **etcd** | control plane | The database holding every object (pods, deployments, secrets). |
| **kube-scheduler** | control plane | Picks a node for each new pod, based on free resources, taints and selectors. |
| **kube-controller-manager** | control plane | Control loops, for example "Deployment wants 2 pods, only 1 exists, so create one". |
| **coredns** | any node (here the control plane) | Cluster DNS, so `mongodb` and `backend-service` resolve to IPs. |
| **kube-proxy** | every node (DaemonSet) | Programs the network rules that make Service IPs work. |
| **kindnet** | every node (DaemonSet) | kind's network plugin (CNI): gives pods IPs and connects them across nodes. |

The four control-plane components are **static pods**. The kubelet on the control plane starts them directly from files on disk, not through the API server:

```powershell
docker exec demo-cluster-control-plane ls /etc/kubernetes/manifests
# etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml

kubectl get pod kube-apiserver-demo-cluster-control-plane -n kube-system -o jsonpath='{.metadata.ownerReferences[0].kind}'
# Node      <- owned by the node itself, not by a Deployment
```

That's why their names end in the node name. Run the `docker exec` commands in this guide from PowerShell: Git Bash rewrites paths like `/etc/...` (prefix the command with `MSYS_NO_PATHCONV=1` there).

## 6. Is the control plane healthy?

```powershell
kubectl get --raw='/readyz?verbose'
# [+]etcd ok
# ...
# readyz check passed

kubectl get --raw='/livez?verbose'      # same idea: "is the API server alive?"
```

Use these instead of the old `kubectl get componentstatuses`, which is deprecated.

**Leader election.** In a cluster with 3 control planes, only one scheduler and one controller manager are active at a time. They take turns through **Leases**:

```powershell
kubectl get lease -n kube-system
# kube-controller-manager   demo-cluster-control-plane_bc66...
# kube-scheduler            demo-cluster-control-plane_461c...
```

The HOLDER column shows which instance is currently in charge. There's only one control plane here, so it's always the same node.

## 7. Is a worker healthy?

```powershell
kubectl describe node demo-cluster-worker
```

The parts worth reading:

```
Conditions:
  MemoryPressure   False   KubeletHasSufficientMemory
  DiskPressure     False   KubeletHasNoDiskPressure
  PIDPressure      False   KubeletHasSufficientPID
  Ready            True    KubeletReady          kubelet is posting ready status

Capacity:     cpu: 16   memory: 8096504Ki   pods: 110
Allocatable:  cpu: 16   memory: 8096504Ki   pods: 110

Allocated resources:
  cpu      220m (1%)    memory  466Mi (5%)
```

- **Conditions:** a pressure condition that isn't `False`, or `Ready` that isn't `True`, means trouble.
- **Capacity / Allocatable:** what the node has, and what is available to pods. They're equal here; on real servers some is reserved for the OS. All kind nodes share your PC, so each one shows the full 16 CPUs and about 8 GiB.
- **Allocated resources:** the sum of the **requests** of the pods on the node. The scheduler uses this to decide whether a pod fits. It is not actual usage.

**Node heartbeats.** Each kubelet renews a Lease every few seconds. If renewals stop for about 40 seconds, the node is marked `NotReady`.

```powershell
kubectl get lease -n kube-node-lease
# demo-cluster-control-plane   demo-cluster-worker   demo-cluster-worker2
```

**The kubelet and container runtime** aren't pods, so kubectl can't list them. Check them inside the node instead:

```powershell
docker exec demo-cluster-worker systemctl is-active kubelet containerd
# active
# active
```

On a real VM you'd SSH in and run the same `systemctl` command.

## 8. What is running on each worker?

```powershell
kubectl get pods -A -o wide --field-selector spec.nodeName=demo-cluster-worker
```
```
demo-namespace   backend-6458c7bc99-rspb8    10.244.1.2
demo-namespace   frontend-594fdb5667-khp2d   10.244.1.3
kube-system      kindnet-g8pdj               172.18.0.5
kube-system      kube-proxy-bk2jc            172.18.0.5
```

```powershell
kubectl get pods -A -o wide --field-selector spec.nodeName=demo-cluster-worker2
```
```
demo-namespace   backend-6458c7bc99-xwd4p    10.244.2.8
demo-namespace   frontend-594fdb5667-lc5z8   10.244.2.3
demo-namespace   mongodb-0                   10.244.2.7
demo-namespace   mongo-client                10.244.2.4
kube-system      kindnet-q8b29               172.18.0.7
kube-system      kube-proxy-ntttl            172.18.0.7
```

Three things to notice:
- The scheduler put one backend and one frontend on each worker, so losing one worker doesn't take the app down.
- Each node has its own **pod IP range**: `10.244.0.x` on the control plane, `10.244.1.x` on worker, `10.244.2.x` on worker2. kindnet routes between them.
- System pods like kube-proxy show the **node's IP** (`172.18.0.x`) because they use the host network.

## 9. Node resource usage

```powershell
kubectl top nodes
# error: Metrics API not available
```

This needs **metrics-server**, which kind doesn't install. Until you add it, `describe node` (section 7) shows requests, not real usage.

## 10. Experiments to see the roles in action

These change the cluster. Run them only when you want to experiment, and undo them afterwards:

```powershell
# Drain a worker: mark it unschedulable and evict its pods; watch them move to worker2
kubectl drain demo-cluster-worker --ignore-daemonsets --delete-emptydir-data
kubectl get pods -n demo-namespace -o wide -w
kubectl uncordon demo-cluster-worker            # allow scheduling again

# Simulate a dead worker: after ~40s it shows NotReady, then pods get rescheduled
docker stop demo-cluster-worker2
kubectl get nodes -w
docker start demo-cluster-worker2
```

`mongodb-0` lives on worker2, and its volume is local-path storage on that node. If worker2 dies, mongodb can't simply move to another node. That's one of the learning-setup limits described in [production.md](production.md).

## Cheat sheet

| Question | Command |
|---|---|
| Which nodes, and which is the control plane? | `kubectl get nodes -o wide` |
| Only control plane / only workers | `kubectl get nodes -l node-role.kubernetes.io/control-plane` / `-l '!node-role.kubernetes.io/control-plane'` |
| Node labels and taints | `kubectl get nodes --show-labels`, `kubectl describe node <name>` |
| Control-plane components | `kubectl get pods -n kube-system -o wide` |
| Control-plane health | `kubectl get --raw='/readyz?verbose'` |
| Who is the leader | `kubectl get lease -n kube-system` |
| Node health and capacity | `kubectl describe node <name>` |
| Node heartbeats | `kubectl get lease -n kube-node-lease` |
| Pods on one node | `kubectl get pods -A -o wide --field-selector spec.nodeName=<name>` |
| Real CPU and memory | `kubectl top nodes` (needs metrics-server) |
