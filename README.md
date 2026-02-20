# claude experiments

A hands-on comparison of how prompt quality affects AI-generated code.

The same task — *"write a Kubernetes Operator in Go that manages Helm chart releases"* — is given twice:

| | Prompt | Result |
|---|---|---|
| [`first/`](./first) | One vague sentence | Missing deps, no DeepCopy, no Helm calls — won't compile |
| [`second/`](./second) | Full design doc | Production-quality operator with web UI, tests, and Makefile |

Read each [`PROMPT.md`](./second/PROMPT.md) before looking at the code.

---

## second/ — Well-Written Prompt Result

A fully functional Kubernetes Operator that installs, upgrades, and uninstalls Helm chart releases via a `HelmRelease` custom resource, with a built-in web UI.

### Prerequisites

- Go 1.21+
- A running Kubernetes cluster (e.g. [kind](https://kind.sigs.k8s.io/), minikube, or any kubeconfig-reachable cluster)
- `kubectl` configured to point at the cluster
- `helm` CLI (for manual verification only)

### Quick Start

```bash
# 1. Install the CRD
kubectl apply -f second/config/crd/bases/helm.example.com_helmreleases.yaml

# 2. Run the operator locally (no Docker required)
cd second
go run ./main.go --leader-elect=false
```

The operator starts three listeners:

| Port | Purpose |
|------|---------|
| `:8080` | Prometheus metrics |
| `:8081` | `/healthz` and `/readyz` probes |
| `:8082` | Web UI |

Open **http://localhost:8082** to use the web UI.

---

## Deploy to Kind (local cluster)

This section covers building a Docker image, loading it into a local [Kind](https://kind.sigs.k8s.io/) cluster, and installing the operator with the Helm chart.

### Additional prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [`kind`](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- `helm` CLI

### 1. Create a Kind cluster

```bash
kind create cluster --name helm-operator-demo
```

### 2. Build the Docker image

```bash
cd second
docker build -t helm-operator:v0.1.0 .
```

### 3. Load the image into Kind

Kind nodes cannot pull from the local Docker daemon automatically, so load the image directly:

```bash
kind load docker-image helm-operator:v0.1.0 --name helm-operator-demo
```

### 4. Install with Helm

```bash
helm install helm-operator ./chart \
  --namespace helm-operator \
  --create-namespace \
  --set image.tag=v0.1.0
```

The chart applies the CRD first (from `chart/crds/`), then deploys the operator Deployment, ServiceAccount, ClusterRole, and ClusterRoleBinding.

Verify the operator is running:

```bash
kubectl get pods -n helm-operator
# NAME                             READY   STATUS    RESTARTS   AGE
# helm-operator-7d9f6b8c4d-xq2pz   1/1     Running   0          30s
```

### 5. Access the web UI from your laptop

```bash
kubectl port-forward -n helm-operator svc/helm-operator-ui 8082:8082
```

Then open **http://localhost:8082**.

---

## Working Example — Deploy nginx via HelmRelease

```bash
kubectl create namespace demo

kubectl apply -f - <<EOF
apiVersion: helm.example.com/v1alpha1
kind: HelmRelease
metadata:
  name: my-nginx
  namespace: demo
spec:
  chart: nginx
  repoURL: https://charts.bitnami.com/bitnami
  version: "15.0.0"
  targetNamespace: demo
  values:
    replicaCount: 1
    service:
      type: ClusterIP
EOF
```

Watch the operator reconcile:

```bash
kubectl get hr -n demo -w
# NAME       CHART   VERSION   NAMESPACE   PHASE        AGE
# my-nginx   nginx   15.0.0    demo        Installing   2s
# my-nginx   nginx   15.0.0    demo        Ready        14s
```

### Tear down

```bash
helm uninstall helm-operator -n helm-operator
kind delete cluster --name helm-operator-demo
```

---

## Web UI

The operator embeds a single-page UI served on `:8082`. No separate deployment is required.

### Features

- **List** all `HelmRelease` resources across namespaces, with colour-coded phase badges
- **Create** a new release via a modal form
- **Edit** an existing release (chart, version, repo URL, values)
- **Delete** a release (the operator's finalizer ensures the Helm release is also uninstalled)
- **Live status updates** via Server-Sent Events — the table refreshes automatically as the operator reconciles changes

### Running with the UI

```bash
cd second
make run-ui          # equivalent to: go run ./main.go --leader-elect=false --ui-bind-address=:8082
```

To change the UI port:

```bash
go run ./main.go --leader-elect=false --ui-bind-address=:9090
```

---

## kubectl Usage

### HelmRelease spec

```yaml
apiVersion: helm.example.com/v1alpha1
kind: HelmRelease
metadata:
  name: <release-name>       # also the Helm release name unless releaseName is set
  namespace: <cr-namespace>  # namespace where this CR lives
spec:
  chart: <chart-name>        # required
  repoURL: <repo-url>        # required
  version: <chart-version>   # required — exact semver (e.g. "15.0.0")
  targetNamespace: <ns>      # required — where the Helm release is installed
  releaseName: <name>        # optional — overrides the Helm release name
  values: {}                 # optional — arbitrary Helm values
```

### Command reference

```bash
# List (short alias: hr)
kubectl get hr                                               # default namespace
kubectl get hr -A                                            # all namespaces
kubectl get hr -n demo -w                                    # watch for phase changes

# Create / update
kubectl apply -f helmrelease.yaml

# Inspect
kubectl describe hr my-nginx -n demo                         # phase, conditions, revision
kubectl get hr my-nginx -n demo -o yaml                      # full resource

# Upgrade — edit spec, operator reconciles automatically (Ready → Upgrading → Ready)
kubectl patch hr my-nginx -n demo --type=merge -p '{"spec":{"version":"15.1.0"}}'
kubectl edit hr my-nginx -n demo

# Delete — finalizer runs helm uninstall before CR is removed
kubectl delete hr my-nginx -n demo

# Operator logs
kubectl logs -n helm-operator -l app.kubernetes.io/name=helm-operator -f

# Events for a specific release
kubectl events --for helmrelease/my-nginx -n demo
```

---

## Project Structure

```
claude-tutorials/
├── README.md
├── first/                        ← poor-prompt version (won't compile)
│   ├── PROMPT.md
│   ├── main.go
│   ├── api/v1alpha1/
│   └── controllers/
└── second/                       ← well-written-prompt version
    ├── PROMPT.md
    ├── Dockerfile                ← multi-stage build (distroless runtime)
    ├── main.go                   ← operator entry point
    ├── Makefile
    ├── api/v1alpha1/
    │   ├── helmrelease_types.go  ← CRD schema
    │   └── zz_generated.deepcopy.go
    ├── chart/                    ← Helm chart for deploying the operator
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   ├── crds/
    │   │   └── helm.example.com_helmreleases.yaml
    │   └── templates/
    │       ├── _helpers.tpl
    │       ├── serviceaccount.yaml
    │       ├── clusterrole.yaml
    │       ├── clusterrolebinding.yaml
    │       ├── deployment.yaml
    │       └── service.yaml
    ├── config/crd/bases/         ← generated CRD (source of truth)
    ├── controllers/
    │   ├── helmrelease_controller.go  ← reconciler
    │   └── helmclient.go              ← Helm SDK wrapper
    └── web/
        ├── server.go             ← HTTP server + SSE broker
        └── static/
            └── index.html        ← embedded single-page UI
```

---

## Makefile Targets (second/)

```bash
make build        # compile binary to bin/manager
make run          # run operator locally (leader election on)
make run-ui       # run with leader election off, UI on :8082
make test         # run unit + integration tests via envtest
make manifests    # regenerate CRD YAML
make generate     # regenerate DeepCopy methods
make fmt          # gofmt
make vet          # go vet
make docker-build # build Docker image (IMG=helm-operator:latest)
make docker-push  # push image to registry
make install      # kubectl apply CRDs
make deploy       # kubectl apply all manifests
```

---

## Key Design Contrasts (first vs second)

| Concern | `first/` (poor prompt) | `second/` (well-written prompt) |
|---------|----------------------|--------------------------------|
| CRD spec fields | `chart`, `repoURL`, `version` | + `targetNamespace`, `releaseName`, `values` |
| Status model | `installed: bool` | `phase`, `conditions`, `deployedVersion`, `helmRevision`, `lastDeployedAt` |
| DeepCopy | Missing — won't compile | Full `zz_generated.deepcopy.go` |
| Helm calls | `fmt.Printf` + TODO | Real `helm.sh/helm/v3/pkg/action` calls |
| Finalizer | None — orphans releases on delete | `helm.example.com/finalizer` |
| RBAC markers | None | Full `+kubebuilder:rbac` markers |
| Error handling | No requeue | `setFailedStatus`, requeues after 30 s |
| Web UI | None | Embedded SPA on `:8082` |
| Tests | None | envtest + Ginkgo suite |
| Makefile | None | Full set of targets |
