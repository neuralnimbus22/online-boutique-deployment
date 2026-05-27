# CLAUDE.md — online-boutique-deployment

## What this is
A single-file Kubernetes manifest for **Google Cloud's microservices-demo (Online Boutique) at v0.10.5**, with one local customization: `cartservice` memory bumped to `requests: 256Mi / limits: 512Mi` (default is lower; the bump prevents OOMKills under load). This repo is the source-of-truth manifest that the broader demo stack uses — the `testkube-tests` Playwright suite, the `predemo-check.sh` sanity script in that repo, and any TestKube workflow targeting Online Boutique all expect this workload to be deployed.

## Architecture
Standard Google Online Boutique topology — 11 microservices + 1 cache + 1 loadgen, communicating over gRPC behind a single `frontend` HTTP service. No local code, no Helm, no Kustomize — just raw manifests.

```
                       ┌──► adservice         (gRPC :9555)
                       ├──► cartservice ──► redis-cart  (Redis :6379)
                       ├──► checkoutservice ──► payment, shipping, email, currency, cart, product
                       ├──► currencyservice   (gRPC :7000)
frontend  ───► gRPC ───┼──► productcatalogservice  (gRPC :3550)
(:80 HTTP)             ├──► recommendationservice  (gRPC :8080)
                       ├──► shippingservice   (gRPC :50051)
                       ├──► paymentservice    (gRPC :50051)
                       └──► emailservice      (gRPC :5000)

loadgenerator (locust) ──► frontend  (synthetic traffic)
frontend-external (LoadBalancer)  ──► frontend  (single external entrypoint)
```
- All services run from upstream images at `us-central1-docker.pkg.dev/google-samples/microservices-demo/<name>:v0.10.5`.
- `redis-cart` uses `redis:alpine`. `loadgenerator` uses `busybox:latest` as an initContainer (wait-for-frontend).
- No namespace is set in the manifest — applied to whatever the current `kubectl` context default is.

## Key directories
| Path | Contents |
|---|---|
| `kubernetes-manifests-fresh.yaml` | The whole stack — 12 Deployments + 12 Services + 11 ServiceAccounts, ~980 lines. |

That's it. No other files.

## How to deploy
**To the active kubectl context (no namespace in manifest — usually `default`):**
```bash
kubectl apply -f kubernetes-manifests-fresh.yaml
```

**To a specific namespace (this is how it's actually used in the demo stack):**
```bash
kubectl create namespace online-boutique
kubectl apply -n online-boutique -f kubernetes-manifests-fresh.yaml
kubectl -n online-boutique rollout status deploy/frontend --timeout=300s
```

**Reach the frontend locally (Docker Desktop / no real LoadBalancer):**
```bash
kubectl -n online-boutique port-forward svc/frontend 8080:80
# then: http://localhost:8080
```

**Tear down:**
```bash
kubectl delete -f kubernetes-manifests-fresh.yaml -n online-boutique
# or: kubectl delete namespace online-boutique
```

## Conventions / gotchas
- **The `online-boutique` namespace is the convention** used by adjacent repos in this demo (the `testkube-tests` Playwright suite expects it, `infra-rbac.yaml` in that repo grants TestKube read access into it). The manifest itself does not declare a namespace — set it at apply time with `-n online-boutique`.
- **One local customization: cartservice memory bump.** If updating to a newer upstream version, re-apply this bump to `cartservice` — without it cartservice OOMKills under loadgen pressure.
- **`frontend-external` is `type: LoadBalancer`** — on Docker Desktop this stays `<pending>`. Port-forward `svc/frontend` (ClusterIP) instead.
- **Images are pulled from Google's public Artifact Registry** — no auth needed but first pull is slow on a cold node.
- **loadgenerator generates real traffic** as soon as the stack is up. If you want a quiet stack for testing, `kubectl scale deploy/loadgenerator --replicas=0`.
- **The full app is grpc-heavy under the hood**; only `frontend` speaks HTTP, so external tests should always target `frontend`.

## Common tasks
- **Re-deploy a clean stack** → `kubectl delete -f kubernetes-manifests-fresh.yaml -n online-boutique` then re-apply. Pods come up in ~30–60s on a warm node.
- **Quiet the loadgen for clean test runs** → `kubectl -n online-boutique scale deploy/loadgenerator --replicas=0` (restore with `--replicas=1` later).
- **Update to a newer upstream version** → grab the new manifests file from https://github.com/GoogleCloudPlatform/microservices-demo, re-apply the cartservice memory bump (requests:256Mi / limits:512Mi), commit.
- **Verify it's healthy** → `kubectl -n online-boutique get pods` (all `1/1 Running`) and the `scripts/predemo-check.sh` in the `testkube-tests` repo covers it as part of its broader sanity sweep.
