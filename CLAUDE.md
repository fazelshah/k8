# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Infrastructure config for a single Amazon EKS cluster (`k8-cluster`, region `ap-south-1`), driven entirely by manually-triggered GitHub Actions. There is no application code, no build system, and no test suite — every file is either an eksctl cluster spec, a Helm values file, a raw Kubernetes manifest, or a workflow that applies one of those to the cluster.

## Architecture

**Workflow ↔ values-file pairing.** Each file in `.github/workflows/` is `workflow_dispatch`-only (nothing runs on push) and installs exactly one thing using a values file from the repo root. The pairing is by convention, not enforced, and the names do not match cleanly:

| Workflow | Values/manifest | Release / namespace |
|---|---|---|
| `create-cluster.yaml` | `create-cluster.yaml` (eksctl ClusterConfig) | — |
| `create-alb.yaml` | `alb.yaml` | `aws-load-balancer-controller` / `kube-system` |
| `create-ingress.yaml` | `ingress-values.yaml` | `ingress-nginx` / `ingress` |
| `create-trafik.yaml` | `trafik.yaml` | `traefik` / `traefik` |
| `create-haproxy.yaml` | `haproxy-values.yaml` | `haproxy` / `haproxy` |
| `create-cert.yaml` | (no values; `--set crds.enabled=true`) | `cert-manager` / `cert-manager` |
| `create-issuer.yaml` | `issuer.yaml` (`kubectl apply`, not Helm) | ClusterIssuer `letsencrypt-prod` |
| `create-grafana.yaml` | `garfana-values.yaml` *(sic)* | `grafana` / `monitoring` |
| `create-influx.yaml` | `influx.yaml` | `influxdb` / `influx` |
| `victoria.yaml` | `victoria-values.yaml` | `victoria-metrics` / `victoria` |
| `create-otel.yaml` | `otel.yaml` **and** `otel-daemon.yaml` | `opentelemetry-collector` / `apm` |
| `newrelic.yaml` (workflow) | `newrelic.yaml` (values — same name!) | `newrelic-bundle` / `newrelic` |
| `create-discourse.yaml` | `discourse.yaml` | `discourse` / `discourse` |
| `create-superset.yaml` | `superset.yaml` | `superset` / `superset` |
| `dora.yaml` | `values.yaml` (Apache DevLake) | `devlake` / `dora` |
| `update.yaml` | — (`eksctl utils update-cluster-vpc-config`) | — |
| `delete.yaml` | — (`eksctl delete cluster`) | — |

**Every deploy workflow follows the same five steps:** checkout → `aws-actions/configure-aws-credentials` with `AWS_EKS_ACCESS_KEY_ID` / `AWS_EKS_SECRET_ACCESS_KEY` → `aws eks update-kubeconfig` → `azure/setup-helm@v4` pinned to `v3.19.2` → `helm upgrade --install`. When adding a component, copy an existing workflow rather than inventing a new shape. Both `eksClusterName` and `awsRegion` are single-option `choice` inputs (`k8-cluster`, `ap-south-1`) and several workflows ignore the region input and hardcode `ap-south-1` anyway.

**Cluster shape** (`create-cluster.yaml`): k8s 1.34, one managed nodegroup of 3× `t3a.medium` pinned to `ap-south-1b`/`ap-south-1c`, NAT gateway disabled, kubelet image-GC thresholds lowered to 70/50. Pod-to-AWS auth uses **EKS Pod Identity, not IRSA** — `iam.podIdentityAssociations` creates the `aws-load-balancer-controller` service account in `kube-system` with the well-known ALB policy, which is why `alb.yaml` sets `serviceAccount.create: false`. Addons: eks-pod-identity-agent, ebs-csi (default StorageClass), vpc-cni, kube-proxy, coredns, metrics-server, and `amazon-cloudwatch-observability`.

**Four ingress controllers coexist** (ALB, nginx, Traefik, HAProxy), each in its own namespace, and workloads pick one via `ingressClassName`. Consequence: changing which controller a service uses means editing that service's values file *and* checking that the controller is actually installed. TLS is split two ways — ALB-fronted apps (`discourse`, `superset`, DevLake) reference a hardcoded ACM certificate ARN in their annotations, while Grafana uses cert-manager + `letsencrypt-prod`. All hostnames are under `fazelshah.fun`.

The `letsencrypt-prod` ClusterIssuer in `issuer.yaml` pins its HTTP-01 solver to `ingressClassName: haproxy`, matching the class Grafana requests. **If you move a cert-manager-backed service to a different ingress controller, change the solver class too** — otherwise ACME challenges are routed to a controller that isn't serving the hostname and issuance silently fails. The `haproxytech/kubernetes-ingress` chart names its IngressClass `haproxy`, so `haproxy` is the correct value for both.

**Observability chain:** the OTel collector (deployment + daemonset) receives OTLP on 4317/4318 and dual-exports to VictoriaMetrics (`prometheusremotewrite` to the in-cluster `vminsert` service) and SigNoz Cloud. Editing the exporter endpoint in `otel.yaml` almost always means editing `otel-daemon.yaml` identically — the two config blocks are duplicated verbatim apart from `mode` and the `hostMetrics`/`kubeletMetrics`/`clusterMetrics` presets. Separately, the `amazon-cloudwatch-observability` addon (configured inline in `create-cluster.yaml`) pushes node memory and root-disk metrics to the `Custom/NodeMetrics` CloudWatch namespace on a 60s interval, with container logs explicitly disabled — so node-level metrics reach both CloudWatch and the OTel path.

**Secrets:** the `create-discourse.yaml` and `newrelic.yaml` workflows install `kubeseal` 0.27.3 and pipe `kubectl create secret --dry-run=client` through it into the cluster, then `kubectl wait --for=jsonpath='{.data}'` for the controller to decrypt. This assumes a `sealed-secrets-controller` in `kube-system` — **no workflow in this repo installs it**, so it must already exist on the cluster. Note that several values files still carry plaintext credentials inline (`discourse.yaml`, `influx.yaml`, `superset.yaml`, the SigNoz ingestion key in both otel files); the sealed-secrets path is the intended direction for anything new.

## Validating changes

There is no lint/test/build tooling configured, and workflows cannot be exercised without a live cluster. Before pushing, validate locally against the upstream chart:

```bash
# Render a values file against its chart (catches schema/indentation errors)
helm repo add grafana https://grafana.github.io/helm-charts && helm repo update
helm template grafana grafana/grafana -f garfana-values.yaml

# Raw manifests
kubectl apply --dry-run=client -f issuer.yaml

# Workflow syntax
actionlint .github/workflows/create-grafana.yaml
```

`helm template` is the meaningful check — a values key that the chart does not recognize is silently ignored, which is the most common failure mode here (see the VictoriaMetrics note below).

## Known rough edges

Do not "clean these up" incidentally; they are load-bearing or unverified. Fix them only when asked.

- **`garfana-values.yaml`'s HAProxy annotations use the wrong prefix.** They are `haproxy-ingress.github.io/*` (the *jcmoraisjr/haproxy-ingress* controller), but the installed controller is `haproxytech/kubernetes-ingress`, which reads `haproxy.org/*` and identifies itself as `haproxy.org/ingress-controller/haproxy`. The `proxy-body-size` and `timeout-*` annotations are therefore silently ignored; only `cert-manager.io/cluster-issuer` takes effect. The same wrong prefix appears in `test-haproxy.yaml`.
- **`victoria-values.yaml` uses `server:` / `persistentVolume:` keys** (the single-node `victoria-metrics-single` chart shape) while `victoria.yaml` installs `vm/victoria-metrics-cluster`. The otel exporters point at a `vminsert` service, i.e. the cluster chart.
- **`create-influx.yaml` and `dora.yaml` run on `runs-on: self-hosted`** while everything else uses `ubuntu-latest`/`ubuntu-22.04`. Recent commits moved other workflows off self-hosted.
- **`update.yaml`'s step is named "Disable public access"** but sets `--public-access=true --public-access-cidrs=0.0.0.0/0`.
- **`create-cluster.yaml` VPC CIDR is `192.169.0.0/16`** with an inline comment intending `192.168.0.0/16` on the next deployment.
- **`test-nginx.yaml` / `test-haproxy.yaml` are untracked scratch files** — Go-templated Helm Ingress templates for an unrelated Cryptlex app chart (`cryptlex-alb-group`, web-api/admin-portal/customer-portal/reseller-portal/filestore services), kept side by side to compare nginx vs HAProxy annotation equivalents. They are not part of any workflow and will not parse as plain YAML.
