# Project 2: Nginx on Kubernetes with YAML Manifests

Deploying an Nginx web server on Kubernetes (minikube) using declarative YAML files, then scaling it and debugging a deliberately broken Service.

## What I built

| File | Resource | Purpose |
|---|---|---|
| `deployment.yaml` | Deployment `web` | Runs `nginx:1.27` and keeps the desired number of Pod replicas running |
| `service.yaml` | Service `web` (ClusterIP) | Gives the Pods one stable address and load balances across them |

The Service finds the Pods through the label `app: web`. The label in the Pod template and the Service selector must match.

## Tools

Kubernetes (minikube), kubectl, YAML, Docker driver, Ubuntu

## How to run it

```bash
# Validate against the API server before applying
kubectl apply --dry-run=server -f deployment.yaml -f service.yaml

# Deploy
kubectl apply -f deployment.yaml -f service.yaml

# Check
kubectl get pods
kubectl get service web
kubectl get endpoints web

# Open the app in a browser at http://localhost:8081
kubectl port-forward service/web 8081:80

# Clean up
kubectl delete -f deployment.yaml -f service.yaml
```

## What I practised

### 1. Fixing YAML errors
My first apply failed with `unknown field` errors. I fixed:
- wrong `apiVersion` (`apps/v1` is needed for a Deployment)
- typos in field names (`matchLabels`, `labels`)
- case-sensitive field names (`targetPort`, not `targetport`)
- indentation: the Pod's `spec:` must sit inside `template:`

Lesson: `--dry-run=client` only checks that the file is valid YAML. `--dry-run=server` also checks the field names against Kubernetes.

### 2. Scaling declaratively
I changed `replicas: 3` to `replicas: 5` in the file and ran `kubectl apply` again. Kubernetes reported `deployment.apps/web configured` and added only the 2 missing Pods, leaving the original 3 untouched.

### 3. Breaking the Service on purpose
I changed the Service selector from `app: web` to `app: wrong` and applied it.

- All 5 Pods stayed `Running` and healthy.
- `kubectl get endpoints web` showed `<none>`.
- `kubectl describe service web` showed `Selector: app=wrong` and empty Endpoints.

The Service could not find any Pods because none carried the label `app=wrong`. This is the classic "Pods are running but the app is unreachable" problem.

### 4. Fixing it
I changed the selector back to `app: web`, re-applied, and the Endpoints showed the Pod IP addresses again.

## Debugging checklist

When an app is Running but unreachable:

```bash
kubectl get pods --show-labels     # what labels do the Pods have?
kubectl describe service web       # what selector does the Service use?
kubectl get endpoints web          # do they match?
```

## Screenshots

| Description | File |
|---|---|
| Successful apply and 3 Pods running | `screenshots/01-apply.png` |
| 5 Pods running after scaling | `screenshots/02-five-pods.png` |
| Broken Service: `Selector: app=wrong`, no Endpoints | `screenshots/03-broken-service.png` |
| Fixed Service with Endpoints restored | `screenshots/04-fixed-endpoints.png` |
| Nginx welcome page in the browser | `screenshots/05-nginx-page.png` |

## Key takeaways

- Labels and selectors are how Kubernetes connects resources.
- The YAML file is the source of truth: change the file, then re-apply.
- Read the error message. The field path it prints (for example `spec.ports[0].targetport`) points to the exact problem.
