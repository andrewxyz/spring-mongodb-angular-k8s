# How this differs from production-grade Kubernetes

Production Kubernetes uses the same objects as this project (Deployments, Services, Ingress, ConfigMaps, Secrets, StatefulSets). The differences are in who runs the cluster, how traffic gets in, where state lives, how changes ship, and how you find out when something breaks.

This repo is a learning setup. Each section below compares it with what a production setup usually does.

## 1. Cluster

| This repo (kind) | Production |
|---|---|
| 1 control plane and 2 workers, all Docker containers on one PC | Managed service (EKS, GKE, AKS) or a self-run cluster with 3+ control-plane nodes |
| One machine, so one failure takes everything down | Nodes spread across 3 availability zones |
| Fixed size | Cluster autoscaler or Karpenter adds and removes nodes |

A managed control plane means you don't run etcd or the API server yourself.

## 2. Getting traffic in

- **Load balancer:** Traefik here uses `hostPort` on one node. In production a cloud load balancer (`Service type: LoadBalancer`) sits in front of several Traefik or NGINX replicas, so there is no single entry node.
- **HTTPS:** none here. Production uses **cert-manager** with Let's Encrypt for automatic certificates, and a real DNS name instead of `mylocal.localhost` (usually managed by **external-dns**).
- **Gateway API:** it is replacing Ingress for new setups. It has richer routing and clearer ownership between platform and app teams.

## 3. Workloads

The Deployments here are close already. Production adds:

- **HorizontalPodAutoscaler:** replicas follow CPU, memory or request load instead of a fixed 2.
- **PodDisruptionBudget:** node upgrades can't evict all replicas at once.
- **Anti-affinity or topology spread:** replicas land on different nodes and zones. In kind they can share a node.
- **Resource requests and limits on everything** (already done here), plus **ResourceQuota** and **LimitRange** per namespace.
- **Rolling update tuning** (`maxUnavailable: 0`) and **graceful shutdown** (`preStop`, `terminationGracePeriodSeconds`), so deploys drop no requests.
- **Pinned image digests or immutable tags** from a private registry, never `latest`.

## 4. State

MongoDB here is 1 replica on a local-path disk. A production database is one of these:

- **A managed database:** Atlas, DocumentDB or Cosmos DB. The most common choice, because running databases in Kubernetes is hard.
- **A database operator:** the MongoDB Community Operator or Percona. It runs a 3-member replica set with automated failover, backups and upgrades.

Either way you also get cloud-backed storage classes (EBS, Persistent Disk) with snapshots, and backups that are restored on a schedule to prove they work.

## 5. Config and secrets

- The password in [mongodb/secret.yaml](mongodb/secret.yaml) is plain text. A base64 Kubernetes Secret is not encryption.
- Production keeps secrets in **Vault, AWS Secrets Manager or Azure Key Vault**. **External Secrets Operator** or the **CSI driver** syncs them into the cluster.
- Alternatives are **Sealed Secrets** or **SOPS**, which let you commit encrypted secrets to git.
- etcd encryption at rest is turned on, and credentials are rotated.

## 6. Security

- **RBAC** with least privilege. Humans don't use cluster-admin day to day.
- **ServiceAccounts** per app, with cloud IAM mapped to them (IRSA or Workload Identity) instead of static keys.
- **NetworkPolicies:** pods here can all talk to each other. Production is default-deny, then allows only frontend → backend → mongodb.
- **Pod Security Standards** (`restricted`): non-root, read-only root filesystem, no privilege escalation, dropped capabilities. The backend image is partly there with its non-root user.
- **Image scanning** (Trivy) and **signing** (cosign) in CI.
- **Admission policies** (Kyverno or OPA Gatekeeper) reject non-compliant manifests.

## 7. Delivery

Here you run `kubectl apply -k k8s/` by hand. Production uses CI plus GitOps:

```
git push → CI: test, build image, scan, push to registry (see dockerhub.md, automated)
        → bump the image tag in a config repo
        → Argo CD or Flux sees the change and syncs it to the cluster
```

- Git is the source of truth, and the cluster is continuously reconciled to match it.
- Separate environments (dev, staging, prod) use Kustomize overlays or Helm values.
- Rollbacks are a `git revert`.
- Deploys can be canary or blue/green (Argo Rollouts, Flagger), with automatic rollback on bad metrics.

See [dockerhub.md](dockerhub.md) for the manual version of the registry step.

## 8. Observability

- **Metrics:** Prometheus and Grafana. Spring Boot Actuator can expose `/actuator/prometheus` for this.
- **Logs:** Loki or an ELK/OpenSearch stack, so logs outlive the pod.
- **Traces:** OpenTelemetry with Tempo or Jaeger.
- **Alerts:** Alertmanager or PagerDuty, tied to SLOs such as error rate and latency, not just "pod down".

## 9. Operations and cost

- Regular Kubernetes version upgrades, node by node, protected by PDBs.
- Multi-cluster or multi-region disaster recovery for critical systems.
- Cost tools (Kubecost) and right-sizing of requests.
- Runbooks and on-call, not just YAML.

## How this project maps

| Already production-style | Learning shortcuts |
|---|---|
| Namespace, Deployments with 2 replicas | Single MongoDB pod |
| Probes, resource requests and limits | Plain-text secret in git |
| ConfigMap/Secret separation | No TLS, no NetworkPolicy |
| Ingress with path routing | Images loaded by hand with `kind load` |
| Non-root backend image | Manual `kubectl apply`, no CI |

## Good next steps to practice in kind

Ideas only; none of these are built in this repo:

- **NetworkPolicy:** needs a CNI that enforces it, since kind's default doesn't.
- **HorizontalPodAutoscaler:** needs metrics-server installed.
- **cert-manager** with a self-signed certificate, to get HTTPS on `mylocal.localhost`.

## Deploying to AWS EKS in production (step by step)

This section walks through running this project on Amazon EKS with the production practices above. It is a guide: none of these files exist in the repo yet, and nothing here has been run against an AWS account.

### What you will build

```
Browser ── https://app.example.com ──► Route 53 (DNS, managed by external-dns)
                                            │
                                            ▼
                         Application Load Balancer (HTTPS, ACM certificate)
                         created by the AWS Load Balancer Controller
                       ┌── /api ────────────┴──────────── / ───┐
                       ▼                                       ▼
              backend pods (2-6, HPA)                  frontend pods (2-6, HPA)
              spread across 3 AZs                      spread across 3 AZs
                       │
                       ▼
              MongoDB Atlas (managed, 3-node replica set, outside the cluster)

Images: Amazon ECR        Secrets: AWS Secrets Manager → External Secrets Operator
Logs/metrics: CloudWatch  Deploys: GitHub Actions → ECR → Argo CD
```

What changes compared with kind:

| kind (this repo) | EKS (this guide) | Why |
|---|---|---|
| Traefik on a `hostPort` | AWS Load Balancer Controller + ALB | A managed, multi-AZ load balancer with built-in TLS via ACM |
| `mylocal.localhost`, no TLS | Your domain in Route 53, ACM certificate | Real DNS and free, auto-renewed certificates |
| MongoDB StatefulSet, 1 pod | MongoDB Atlas | Replication, backups and failover are handled for you |
| Secret YAML in git | Secrets Manager + External Secrets Operator | The password never touches git |
| `kind load docker-image` | Amazon ECR | Nodes pull images from a registry |
| Fixed 2 replicas | HPA, PDB, spread across zones | Scales with load, survives a zone or node failure |
| `kubectl apply` by hand | GitHub Actions + Argo CD | Every change is reviewed, built and deployed the same way |

The examples use these placeholders. Replace them everywhere:

| Placeholder | Meaning |
|---|---|
| `ap-southeast-1` | Your AWS region |
| `<account-id>` | Your 12-digit AWS account ID |
| `belajar-prod` | The cluster name |
| `app.example.com` / `example.com` | Your app's hostname and its Route 53 hosted zone |

### Step 1: Prepare your tools and AWS account

1. Install the [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html), [eksctl](https://eksctl.io/installation/), `kubectl` and `helm`.
2. Log in with an IAM identity that can create EKS clusters, VPCs and IAM roles (an admin role for the initial setup):
   ```bash
   aws configure sso               # or: aws configure
   aws sts get-caller-identity     # shows the account you are in
   ```
3. Make sure your domain has a **public hosted zone in Route 53** (`example.com`). If the domain is registered elsewhere, point its name servers to the hosted zone.
4. Set the shell variables used below:
   ```bash
   export AWS_REGION=ap-southeast-1
   export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
   export REGISTRY=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
   ```

### Step 2: Push the images to Amazon ECR

ECR is AWS's private registry. EKS nodes can pull from it without an extra pull secret (eksctl gives the node role read access).

```bash
# One repository per image; IMMUTABLE stops a tag from being overwritten
aws ecr create-repository --repository-name belajar-backend  --image-tag-mutability IMMUTABLE --image-scanning-configuration scanOnPush=true
aws ecr create-repository --repository-name belajar-frontend --image-tag-mutability IMMUTABLE --image-scanning-configuration scanOnPush=true

# Log Docker in to ECR (the login lasts 12 hours)
aws ecr get-login-password | docker login --username AWS --password-stdin $REGISTRY

# Build for amd64 (matches the node type in step 3), then push
docker build --platform linux/amd64 -t $REGISTRY/belajar-backend:1.0.0  spring-boot-data-mongodb
docker build --platform linux/amd64 -t $REGISTRY/belajar-frontend:1.0.0 angular-crud
docker push $REGISTRY/belajar-backend:1.0.0
docker push $REGISTRY/belajar-frontend:1.0.0
```

Later (step 13) CI does this for you and tags each image with the git commit SHA.

### Step 3: Create the cluster

Write the cluster as a file so it can be reviewed and recreated. Save it as `k8s/eks/cluster.yaml`:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: belajar-prod
  region: ap-southeast-1
  version: "1.34"            # use the newest version EKS supports: aws eks describe-cluster-versions

# 3 availability zones; one NAT gateway per zone so a zone outage doesn't cut outbound traffic
availabilityZones: [ap-southeast-1a, ap-southeast-1b, ap-southeast-1c]
vpc:
  nat:
    gateway: HighlyAvailable

# Manage who can use kubectl with EKS access entries (not the old aws-auth ConfigMap)
accessConfig:
  authenticationMode: API

addons:
  - name: vpc-cni
    configurationValues: '{"enableNetworkPolicy": "true"}'   # enforces NetworkPolicies (step 10)
  - name: coredns
  - name: kube-proxy
  - name: eks-pod-identity-agent                              # lets pods get IAM roles

managedNodeGroups:
  - name: general
    instanceType: m6i.large  # amd64, 2 vCPU / 8 GiB
    minSize: 2
    desiredCapacity: 3
    maxSize: 6
    privateNetworking: true  # nodes get no public IPs
    volumeSize: 50

# IAM roles for the controllers installed in step 4 (EKS Pod Identity)
iam:
  podIdentityAssociations:
    - namespace: kube-system
      serviceAccountName: aws-load-balancer-controller
      wellKnownPolicies:
        awsLoadBalancerController: true
    - namespace: kube-system
      serviceAccountName: external-dns
      wellKnownPolicies:
        externalDNS: true
    - namespace: external-secrets
      serviceAccountName: external-secrets
      permissionPolicy:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Action: [secretsmanager:GetSecretValue, secretsmanager:DescribeSecret]
            Resource: arn:aws:secretsmanager:ap-southeast-1:<account-id>:secret:belajar/prod/*
```

Create it (about 15–20 minutes) and check:

```bash
eksctl create cluster -f k8s/eks/cluster.yaml
kubectl get nodes -o wide          # 3 nodes, one per zone, all Ready
```

eksctl writes the kubeconfig for you. The managed node group does not resize itself; see step 11.

### Step 4: Install the cluster controllers

These are installed once per cluster, not once per app.

**AWS Load Balancer Controller** (turns Ingress objects into ALBs):

```bash
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=belajar-prod \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=$AWS_REGION \
  --set vpcId=$(aws eks describe-cluster --name belajar-prod --query cluster.resourcesVpcConfig.vpcId --output text)
```

**external-dns** (creates the Route 53 record for the Ingress host):

```bash
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm install external-dns external-dns/external-dns -n kube-system \
  --set provider.name=aws \
  --set serviceAccount.name=external-dns \
  --set "domainFilters[0]=example.com" \
  --set policy=upsert-only \
  --set txtOwnerId=belajar-prod
```

`upsert-only` means external-dns never deletes DNS records, so it can't wipe records that something else created.

**External Secrets Operator** (copies secrets from Secrets Manager into Kubernetes Secrets):

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```

**metrics-server** (the HorizontalPodAutoscaler needs it):

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm install metrics-server metrics-server/metrics-server -n kube-system
```

Check that everything is running:

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
kubectl get pods -n kube-system -l app.kubernetes.io/name=external-dns
kubectl get pods -n external-secrets
kubectl top nodes                    # works once metrics-server is ready
```

Traefik is not installed: on EKS the ALB does its job.

### Step 5: Request a TLS certificate (ACM)

```bash
aws acm request-certificate --domain-name app.example.com --validation-method DNS
```

In the AWS console, open **Certificate Manager → the certificate → Create records in Route 53**. After a few minutes the status becomes `Issued`. Note the certificate ARN; the Ingress uses it in step 8. ACM renews the certificate automatically.

### Step 6: Set up the database (MongoDB Atlas)

Running MongoDB yourself in production means handling replication, backups, upgrades and failover. Atlas does that for you, and it runs on AWS.

1. Create an Atlas project and a **dedicated cluster (M10 or higher)** on AWS, **in the same region** as EKS. This gives a 3-node replica set with automated backups.
2. **Database Access:** create a user (for example `belajar-app`) with read/write access to `belajar_db` only. Use a long generated password.
3. **Network Access:** allow the cluster's NAT gateway IPs (pods reach the internet through them):
   ```bash
   VPC_ID=$(aws eks describe-cluster --name belajar-prod --query cluster.resourcesVpcConfig.vpcId --output text)
   aws ec2 describe-nat-gateways --filter Name=vpc-id,Values=$VPC_ID \
     --query 'NatGateways[].NatGatewayAddresses[].PublicIp' --output text
   ```
   Add each IP to the Atlas IP access list. Later, switch to **AWS PrivateLink** so database traffic stays on AWS's private network.
4. Copy the connection string (**Connect → Drivers**). It looks like:
   `mongodb+srv://belajar-app:<password>@<cluster>.mongodb.net/belajar_db?retryWrites=true&w=majority`

Amazon DocumentDB is an AWS-native alternative, but it is only MongoDB-*compatible*. Test the app against it before choosing it.

### Step 7: Store the connection string in Secrets Manager

```bash
aws secretsmanager create-secret --name belajar/prod/mongodb-uri \
  --secret-string 'mongodb+srv://belajar-app:<password>@<cluster>.mongodb.net/belajar_db?retryWrites=true&w=majority'
```

This command, including the password, ends up in your shell history. Clear the history afterwards, or create the secret in the console instead. The secret name matches the `belajar/prod/*` permission given to External Secrets in step 3.

### Step 8: Create a Kustomize overlay for EKS

An **overlay** reuses the existing manifests in `k8s/` and changes only what is different on EKS, so the kind setup keeps working unchanged. Create this folder:

```
k8s/eks/
├── cluster.yaml                 (step 3)
└── overlay/
    ├── kustomization.yaml
    ├── cluster-secret-store.yaml
    ├── external-secret.yaml
    ├── ingress-patch.yaml
    ├── backend-patch.yaml
    ├── frontend-patch.yaml
    ├── hpa.yaml
    ├── pdb.yaml
    └── networkpolicy.yaml       (step 10)
```

**`kustomization.yaml`**: start from `k8s/`, remove what Atlas and the ALB replace, and point the images at ECR.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: demo-namespace

resources:
  - ../..                         # everything in k8s/kustomization.yaml
  - cluster-secret-store.yaml
  - external-secret.yaml
  - hpa.yaml
  - pdb.yaml
  - networkpolicy.yaml

images:
  - name: belajar-backend
    newName: <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/belajar-backend
    newTag: "1.0.0"
  - name: belajar-frontend
    newName: <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/belajar-frontend
    newTag: "1.0.0"

patches:
  # Atlas replaces the in-cluster MongoDB, its Secret and the debug pod
  - patch: |-
      $patch: delete
      apiVersion: apps/v1
      kind: StatefulSet
      metadata:
        name: mongodb
  - patch: |-
      $patch: delete
      apiVersion: v1
      kind: Service
      metadata:
        name: mongodb
  - patch: |-
      $patch: delete
      apiVersion: v1
      kind: Secret
      metadata:
        name: mongodb-secret
  - patch: |-
      $patch: delete
      apiVersion: v1
      kind: Pod
      metadata:
        name: mongo-client
  # The connection string now carries the host, port, database and credentials
  - patch: |-
      $patch: delete
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: backend-config
  # ALB health check for the backend's target group
  - patch: |-
      apiVersion: v1
      kind: Service
      metadata:
        name: backend-service
        annotations:
          alb.ingress.kubernetes.io/healthcheck-path: /actuator/health/readiness
  - path: ingress-patch.yaml
    target: {kind: Ingress, name: demo-ingress}
  - path: backend-patch.yaml
    target: {kind: Deployment, name: backend}
  - path: frontend-patch.yaml
    target: {kind: Deployment, name: frontend}
```

**`cluster-secret-store.yaml`**: tells External Secrets where to read from. With Pod Identity, no credentials are needed here.

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
```

**`external-secret.yaml`**: creates a normal Kubernetes Secret `backend-db` with the key `uri`, refreshed every hour so a rotated password reaches the cluster.

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: backend-db
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: aws-secrets-manager
  target:
    name: backend-db
  data:
    - secretKey: uri
      remoteRef:
        key: belajar/prod/mongodb-uri
```

**`ingress-patch.yaml`**: switches `demo-ingress` from Traefik to an internet-facing ALB with HTTPS.

```yaml
- op: replace
  path: /spec/ingressClassName
  value: alb
- op: replace
  path: /spec/rules/0/host
  value: app.example.com
- op: add
  path: /metadata/annotations
  value:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip        # the ALB sends traffic straight to pod IPs
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"    # HTTP → HTTPS
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-southeast-1:<account-id>:certificate/<id>
```

The `/api` and `/` rules stay the same, so the frontend still calls `/api/tutorials` on its own address.

**`backend-patch.yaml`**: one env var from the new Secret replaces the ConfigMap and the old Secret. It also lets the HPA own the replica count, spreads pods across zones and tightens the container.

```yaml
- op: remove
  path: /spec/replicas                       # the HPA sets this; otherwise every apply resets it to 2
- op: remove
  path: /spec/template/spec/containers/0/envFrom
- op: replace
  path: /spec/template/spec/containers/0/env
  value:
    - name: SPRING_DATA_MONGODB_URI
      valueFrom:
        secretKeyRef:
          name: backend-db
          key: uri
- op: add
  path: /spec/template/spec/topologySpreadConstraints
  value:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app: backend
- op: add
  path: /spec/template/spec/containers/0/securityContext
  value:
    runAsNonRoot: true                       # the image already runs as uid 10001
    allowPrivilegeEscalation: false
    capabilities:
      drop: [ALL]
```

**`frontend-patch.yaml`**: the same idea for the frontend. The stock nginx image starts as root, so `runAsNonRoot` stays out until the image moves to `nginxinc/nginx-unprivileged` (which listens on 8080).

```yaml
- op: remove
  path: /spec/replicas
- op: add
  path: /spec/template/spec/topologySpreadConstraints
  value:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app: frontend
```

**`hpa.yaml`**: 2 to 6 pods each, scaling on CPU. Utilization is measured against the CPU *request*, so set realistic requests first. The backend's 100m is low for a JVM under load.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend
spec:
  scaleTargetRef: {apiVersion: apps/v1, kind: Deployment, name: backend}
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: {type: Utilization, averageUtilization: 70}
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend
spec:
  scaleTargetRef: {apiVersion: apps/v1, kind: Deployment, name: frontend}
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: {type: Utilization, averageUtilization: 70}
```

**`pdb.yaml`**: during node upgrades or scale-down, at least one pod of each app keeps running.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: backend
spec:
  minAvailable: 1
  selector: {matchLabels: {app: backend}}
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: frontend
spec:
  minAvailable: 1
  selector: {matchLabels: {app: frontend}}
```

Preview the result before applying anything:

```bash
kubectl kustomize k8s/eks/overlay
```

Check that there is no `mongodb` StatefulSet, that the images point to ECR, and that the Ingress has `ingressClassName: alb`.

### Step 9: Deploy and verify

```bash
kubectl apply -k k8s/eks/overlay

kubectl get externalsecret -n demo-namespace          # STATUS: SecretSynced
kubectl rollout status deploy/backend  -n demo-namespace
kubectl rollout status deploy/frontend -n demo-namespace
kubectl get ingress -n demo-namespace                 # ADDRESS: the ALB's DNS name (after 2-3 min)
```

The ALB and the DNS record take a few minutes. Then:

```bash
curl -I http://app.example.com                 # 301 redirect to https
curl https://app.example.com/api/tutorials     # JSON from the backend
```

Open `https://app.example.com` in the browser.

If something is stuck:
- `kubectl describe ingress demo-ingress -n demo-namespace` shows ALB events and errors (often a wrong certificate ARN).
- `kubectl logs -n kube-system deploy/aws-load-balancer-controller` shows the controller's own errors.
- `kubectl logs -n demo-namespace deploy/backend`: a timeout to Atlas usually means the NAT IPs aren't in the Atlas access list.
- In the EC2 console, under **Target Groups**, both target groups should show healthy targets.

### Step 10: Lock down pod-to-pod traffic (NetworkPolicy)

The VPC CNI enforces NetworkPolicies because of `enableNetworkPolicy` in step 3. Deny all incoming traffic by default, then allow only the ALB to reach the two apps. With `target-type: ip` the ALB connects from inside the VPC, so allow the VPC CIDR. eksctl's default is `192.168.0.0/16`; check yours with `aws ec2 describe-vpcs`.

**`networkpolicy.yaml`**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}               # every pod in the namespace
  policyTypes: [Ingress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-alb-to-frontend
spec:
  podSelector: {matchLabels: {app: frontend}}
  policyTypes: [Ingress]
  ingress:
    - from: [{ipBlock: {cidr: 192.168.0.0/16}}]
      ports: [{port: 80}]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-alb-to-backend
spec:
  podSelector: {matchLabels: {app: backend}}
  policyTypes: [Ingress]
  ingress:
    - from: [{ipBlock: {cidr: 192.168.0.0/16}}]
      ports: [{port: 8080}]
```

Outgoing traffic (backend → Atlas) is not restricted here. Restrict it too once you know every destination the app needs, including DNS.

### Step 11: Scale the nodes automatically

The HPA adds pods, but if the nodes are full, new pods stay `Pending`. Install a node autoscaler:

- **Karpenter** (recommended on EKS) launches right-sized EC2 instances in seconds and removes empty ones. Follow the [Karpenter getting-started guide](https://karpenter.sh/docs/getting-started/).
- **Cluster Autoscaler** is simpler: it resizes the managed node group between `minSize` and `maxSize`.

### Step 12: Logs, metrics and alerts

The quickest start is the CloudWatch add-on. It ships container logs (Fluent Bit) and metrics (Container Insights):

```bash
aws eks create-addon --cluster-name belajar-prod --addon-name amazon-cloudwatch-observability
```

It needs the `CloudWatchAgentServerPolicy`. Give it through a Pod Identity association for the `cloudwatch-agent` service account in the `amazon-cloudwatch` namespace.

Then:
- Create CloudWatch alarms on ALB 5xx errors, target response time and unhealthy host count. Send them to email, Slack or PagerDuty through SNS.
- Turn on Atlas alerts for connections, disk and replication lag.

When you outgrow CloudWatch, the next step is Amazon Managed Service for Prometheus with Amazon Managed Grafana. Spring Boot can expose `/actuator/prometheus` with the `micrometer-registry-prometheus` dependency.

### Step 13: Automate builds and deploys (CI + GitOps)

**CI with GitHub Actions** builds, pushes and bumps the image tag on every push to `main`. It logs in to AWS through OIDC, so no AWS keys are stored in GitHub. Create an IAM role that trusts GitHub's OIDC provider for your repo and allows pushing to ECR, then add a workflow:

```yaml
# .github/workflows/deploy.yml (backend shown; the frontend job is the same)
name: build-and-release
on:
  push:
    branches: [main]
permissions:
  id-token: write       # for AWS OIDC
  contents: write       # to commit the new image tag
jobs:
  backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<account-id>:role/github-actions-ecr-push
          aws-region: ap-southeast-1
      - id: ecr
        uses: aws-actions/amazon-ecr-login@v2
      - name: Build and push
        run: |
          IMAGE=${{ steps.ecr.outputs.registry }}/belajar-backend:${{ github.sha }}
          docker build -t $IMAGE spring-boot-data-mongodb
          docker push $IMAGE
      - name: Bump the tag in the overlay
        run: |
          cd k8s/eks/overlay
          kustomize edit set image belajar-backend=${{ steps.ecr.outputs.registry }}/belajar-backend:${{ github.sha }}
          git config user.name "github-actions"
          git config user.email "github-actions@users.noreply.github.com"
          git commit -am "deploy backend ${{ github.sha }}"
          git push
```

**CD with Argo CD** watches `k8s/eks/overlay` in git and applies every change, so nobody runs `kubectl apply` against production by hand:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

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
    path: k8s/eks/overlay
  destination:
    server: https://kubernetes.default.svc
    namespace: demo-namespace
  syncPolicy:
    automated:
      prune: true        # delete objects removed from git
      selfHeal: true     # undo manual changes in the cluster
```

A rollback is now a `git revert` of the tag-bump commit. For a staging environment, add a second overlay (`k8s/eks/overlay-staging`) and a second Application.

### Step 14: Give your team access

Access entries map IAM roles to Kubernetes permissions. For example, to give a developer role read-only access to the app's namespace:

```bash
aws eks create-access-entry --cluster-name belajar-prod \
  --principal-arn arn:aws:iam::<account-id>:role/Developers
aws eks associate-access-policy --cluster-name belajar-prod \
  --principal-arn arn:aws:iam::<account-id>:role/Developers \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=demo-namespace
```

Keep cluster-admin for a small emergency ("break-glass") role. Day-to-day changes go through git and Argo CD.

### Step 15: Day-2 operations

- **Upgrades:** EKS supports each Kubernetes version for a limited time; after that, paid extended support applies. Upgrade the control plane first (`eksctl upgrade cluster`), then the add-ons, then the node groups. The PDBs from step 8 keep the app up while nodes are replaced.
- **Backups:** Atlas backs up the data, and the cluster itself can be rebuilt from git (`cluster.yaml` plus the overlay). Practise a rebuild at least once.
- **Secret rotation:** change the Atlas password and update the Secrets Manager value. Once External Secrets has refreshed the Secret, restart the backend with `kubectl rollout restart deploy/backend`.
- **Image updates:** rebuild regularly to pick up base-image security fixes; ECR scan-on-push reports known vulnerabilities.

### Cost

This setup is not free. The main costs are:
- the EKS control plane, about $0.10 per hour (about $73 per month)
- 3 EC2 nodes and their disks
- 3 NAT gateways, charged per hour and per GB of traffic
- the ALB, charged per hour and by traffic
- the Atlas cluster
- CloudWatch logs and metrics

Use the [AWS Pricing Calculator](https://calculator.aws/) for your region. While learning, you can save money with a single NAT gateway (`gateway: Single`), smaller nodes and an Atlas M0 free cluster. Those choices are *not* production-grade.

### Tearing it all down

Delete in this order, so the load balancer controller removes the ALB before the cluster (and the controller) is gone:

```bash
kubectl delete -k k8s/eks/overlay                   # the controller removes the ALB
kubectl get ingress -A                              # wait until none are left
eksctl delete cluster -f k8s/eks/cluster.yaml       # nodes, VPC, NAT gateways, IAM roles
aws ecr delete-repository --repository-name belajar-backend  --force
aws ecr delete-repository --repository-name belajar-frontend --force
aws secretsmanager delete-secret --secret-id belajar/prod/mongodb-uri
```

Because of `upsert-only`, external-dns may leave the Route 53 record behind. If it is still there, delete it in the Route 53 console. Finally, delete the ACM certificate in the console and terminate the Atlas cluster.
