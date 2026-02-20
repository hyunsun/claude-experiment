# claude-tutorials

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

### HelmRelease CRD fields

```yaml
apiVersion: helm.example.com/v1alpha1
kind: HelmRelease
metadata:
  name: <release-name>       # also used as the Helm release name unless releaseName is set
  namespace: <cr-namespace>  # namespace where this CR lives
spec:
  chart: <chart-name>        # required — Helm chart name (e.g. "nginx")
  repoURL: <repo-url>        # required — Helm repo URL (e.g. "https://charts.bitnami.com/bitnami")
  version: <chart-version>   # required — exact chart version (e.g. "15.0.0")
  targetNamespace: <ns>      # required — Kubernetes namespace for the Helm release
  releaseName: <name>        # optional — overrides the Helm release name
  values: {}                 # optional — arbitrary Helm values as a JSON/YAML object
```

### Create a release

```bash
kubectl apply -f - <<EOF
apiVersion: helm.example.com/v1alpha1
kind: HelmRelease
metadata:
  name: my-nginx
  namespace: default
spec:
  chart: nginx
  repoURL: https://charts.bitnami.com/bitnami
  version: "15.0.0"
  targetNamespace: default
  values:
    replicaCount: 2
    service:
      type: ClusterIP
EOF
```

### List releases

```bash
kubectl get helmreleases          # or: kubectl get hr
kubectl get hr -A                 # all namespaces
```

Example output:

```
NAME       CHART   VERSION   NAMESPACE   PHASE   AGE
my-nginx   nginx   15.0.0    default     Ready   2m
```

### Inspect a release

```bash
kubectl describe hr my-nginx
```

Key status fields:

```
Status:
  Phase:             Ready
  Deployed Version:  15.0.0
  Helm Revision:     1
  Last Deployed At:  2026-02-19T10:00:00Z
  Conditions:
    Type:    Ready
    Status:  True
    Reason:  ReconcileSuccess
```

### Upgrade a release

Edit the spec (e.g. bump the version or change values) and apply:

```bash
kubectl patch hr my-nginx --type=merge -p '{"spec":{"version":"15.1.0"}}'
```

Or edit interactively:

```bash
kubectl edit hr my-nginx
```

The operator detects the generation change and runs `helm upgrade` automatically. The phase transitions `Ready → Upgrading → Ready`.

### Delete a release

```bash
kubectl delete hr my-nginx
```

The operator's finalizer (`helm.example.com/finalizer`) runs `helm uninstall` before the CR is removed, preventing orphaned Helm releases.

### Watch reconciliation in real time

```bash
kubectl get hr -w
```

```
NAME       CHART   VERSION   NAMESPACE   PHASE        AGE
my-nginx   nginx   15.0.0    default     Installing   0s
my-nginx   nginx   15.0.0    default     Ready        8s
```

### Common kubectl flags

```bash
# Short name alias
kubectl get hr

# Filter by namespace
kubectl get hr -n production

# Output full YAML
kubectl get hr my-nginx -o yaml

# Show events related to a release
kubectl events --for helmrelease/my-nginx

# Check operator logs
kubectl logs -l control-plane=controller-manager -f
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
    ├── main.go                   ← operator entry point
    ├── Makefile
    ├── api/v1alpha1/
    │   ├── helmrelease_types.go  ← CRD schema
    │   └── zz_generated.deepcopy.go
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
