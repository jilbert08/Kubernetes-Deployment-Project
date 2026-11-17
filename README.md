# Kubernetes Deployments – Scaling, Rolling Updates and Rollback

## Project Objective
Deploy an NGINX web application on Kubernetes with a declarative **Deployment** (3 replicas), expose it with a **Service**, scale it, perform a **zero-downtime rolling update** by changing the image tag, track the rollout, and **roll back** to the previous version.

## Technologies
- Windows, Docker Desktop, Minikube
- Kubernetes, kubectl
- NGINX (`nginx:1.25` → `nginx:1.27`)
- VS Code, PowerShell, Git, GitHub

## Project Structure
```
Kubernetes-Deployment-Project/
├── manifests/
│   ├── deployment.yaml    # Deployment: 3 replicas, resources, rolling update strategy
│   └── service.yaml       # NodePort Service exposing port 80
├── docs/
│   └── rollout-log.txt    # Terminal log: apply, scale, rolling update, rollback
└── README.md
```

## Architecture
```
            Browser / curl
                  │  NodePort 30080
                  ▼
        ┌──────────────────────┐
        │ Service: web-service │  selector: app=web
        └──────────┬───────────┘
        ┌──────────┼───────────┐
        ▼          ▼           ▼
      Pod        Pod         Pod      ◄── ReplicaSet (one per image version)
     nginx      nginx       nginx     ◄── Deployment: web-deployment (replicas: 3)
```

## Kubernetes Concepts

| Concept | Meaning |
|---|---|
| **Pod** | Smallest unit in Kubernetes; runs one or more containers |
| **ReplicaSet** | Keeps the desired number of identical Pods running |
| **Deployment** | Manages ReplicaSets; handles scaling, updates and rollback |
| **Declarative manifest** | A YAML file describing the desired state; `kubectl apply` makes the cluster match it |
| **Labels / Selectors** | `app: web` labels the Pods; the Deployment and Service select Pods by this label |
| **Service** | Stable network address that load-balances traffic to the selected Pods |
| **NodePort** | Exposes the Service on a port of the node (30080) |
| **Readiness probe** | Kubernetes sends traffic to a Pod only after it answers HTTP on port 80 |
| **Resource requests / limits** | Minimum guaranteed and maximum allowed CPU and memory per container |

### Deployment → ReplicaSet → Pods
When the Deployment is created, it creates a **ReplicaSet**, which creates **3 Pods**. When the Pod template changes (for example the image tag), the Deployment creates a **new ReplicaSet** and moves Pods from the old one to the new one step by step. The old ReplicaSet is kept (scaled to 0) so the change can be rolled back.

## Rolling Update Strategies

| Strategy | How it works | Downtime |
|---|---|---|
| **RollingUpdate** (used here) | Replaces Pods gradually: new Pods start, become Ready, then old Pods stop | None |
| **Recreate** | Stops all old Pods first, then starts new ones | Yes |

Settings used in `deployment.yaml`:

| Setting | Value | Meaning |
|---|---|---|
| `maxSurge` | 1 | At most 1 extra Pod above the 3 replicas during the update |
| `maxUnavailable` | 0 | Never fewer than 3 ready Pods → zero downtime |
| `minReadySeconds` | 5 | A new Pod must stay Ready 5 s before it counts as available |
| `readinessProbe` | HTTP GET `/` on 80 | Traffic only goes to Pods that respond |
| `revisionHistoryLimit` | 5 | Keep 5 old ReplicaSets for rollback |
| `kubernetes.io/change-cause` | annotation | Description shown in `kubectl rollout history` |

**Update sequence (3 replicas, maxSurge 1, maxUnavailable 0):** start 1 new Pod → wait until Ready → stop 1 old Pod → repeat until all 3 Pods run the new version.

## Manifests

### `manifests/deployment.yaml`
- `replicas: 3` – three NGINX Pods
- `image: nginx:1.25` (updated to `nginx:1.27` for the rolling update)
- `resources.requests`: 50m CPU, 64Mi memory · `limits`: 200m CPU, 128Mi memory
- `strategy: RollingUpdate` with `maxSurge: 1`, `maxUnavailable: 0`
- labels `app: web` on the template, matched by `selector.matchLabels`

### `manifests/service.yaml`
- `type: NodePort`, `port: 80` → `targetPort: 80`, `nodePort: 30080`
- `selector: app: web` – matches the Deployment's Pod labels

## Steps Performed

```powershell
# 1. Deploy
kubectl apply -f manifests/deployment.yaml
kubectl get deployments
kubectl get rs
kubectl get pods -o wide

# 2. Expose
kubectl apply -f manifests/service.yaml
kubectl get svc web-service
kubectl get endpoints web-service

# 3. Scale
kubectl scale deployment web-deployment --replicas=5
kubectl get pods
kubectl scale deployment web-deployment --replicas=3

# 4. Rolling update (image nginx:1.25 -> nginx:1.27 edited in deployment.yaml)
kubectl apply -f manifests/deployment.yaml
kubectl rollout status deployment/web-deployment
kubectl get rs
kubectl rollout history deployment/web-deployment

# 5. Rollback
kubectl rollout undo deployment/web-deployment
kubectl rollout status deployment/web-deployment
kubectl rollout history deployment/web-deployment
```

The complete terminal output is in [`docs/rollout-log.txt`](docs/rollout-log.txt).

## Results
- 3 Pods were created by one ReplicaSet and became Ready.
- The Service selected all Pods (3 endpoints) and served NGINX on NodePort 30080.
- Scaling to 5 and back to 3 replicas worked instantly.
- The rolling update created a new ReplicaSet and replaced the Pods one at a time; there were always 3 ready Pods, so there was no downtime.
- `kubectl rollout undo` restored `nginx:1.25` using the old ReplicaSet.

## Useful Commands

| Command | Purpose |
|---|---|
| `kubectl apply -f <file>` | Create or update from a manifest |
| `kubectl get deploy,rs,pods` | Show Deployment, ReplicaSets and Pods |
| `kubectl describe deployment web-deployment` | Details and events |
| `kubectl scale deployment web-deployment --replicas=N` | Change the number of replicas |
| `kubectl rollout status deployment/web-deployment` | Watch an update until it finishes |
| `kubectl rollout history deployment/web-deployment` | List revisions |
| `kubectl rollout undo deployment/web-deployment` | Roll back to the previous revision |
| `kubectl rollout undo deployment/web-deployment --to-revision=N` | Roll back to a specific revision |

## Cleanup
```powershell
kubectl delete -f manifests/
```

## Author
**Jilbert Ekka** – RHCSA Certified | Linux & DevOps
