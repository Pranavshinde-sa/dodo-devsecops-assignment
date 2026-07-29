# Dodo Payments — DevSecOps Technical Assessment

Hardening `ledger-api`, a Flask-based payments microservice handling PAN tokenisation
and transaction data, from an insecure baseline into a production-grade, PCI
DSS-aware deployment — across workload hardening, secure CI/CD & supply chain,
zero-trust networking, and offensive recon/pentest.

## Repo layout

```
.
├── app/                          # ledger-api Flask service + Dockerfile
├── task-1/                       # Deploy & harden the workload (K8s manifests, Kyverno policies)
├── task-2/                       # GitOps — ArgoCD Application (pipeline lives in .github/workflows/)
├── task-3/                       # Istio service mesh / zero-trust           — not yet started
├── task-4/                       # Recon & penetration test report            — not yet started
└── .github/workflows/            # CI/CD: SAST, scans, signing, provenance, GitOps deploy
```

## Application under test

`ledger-api` (Flask, `app/app.py`) exposes:

| Method | Path | Description |
|---|---|---|
| GET | `/health` | Liveness/readiness check |
| POST | `/tokenize` | `{"pan": "..."}` → SHA-256-derived opaque token |
| GET | `/transactions` | In-memory transaction records |
| POST | `/import` | Parses a YAML config blob (`yaml.safe_load`, not the unsafe loader) |
| GET | `/fetch?url=` | Server-side fetch of a remote URL, restricted to HTTPS + an `ALLOWED_HOSTS` allowlist + no redirects, as an SSRF mitigation |

Runs as a non-root user (`uid 10001`) baked into the image itself (`Dockerfile`), on
top of `python:3.11-slim`.

## Tasks

| Task | Folder | Summary | Status |
|---|---|---|---|
| 1. Deploy & Harden the Workload | [`task-1/`](./task-1/README.md) | Hardened `ledger-api` + `reporting` neighbour on a 3-node `kind` cluster: non-root, read-only rootfs, dropped capabilities, seccomp, dedicated RBAC, Sealed Secrets, Kyverno guardrails | ✅ |
| 2. Secure CI/CD Pipeline & Supply Chain | [`task-2/`](./task-2/README.md) | GitHub Actions pipeline: Semgrep SAST, Trivy dependency + image scans, gitleaks, GHCR build/push, keyless Cosign signing + verification, SLSA provenance attestation, ArgoCD GitOps auto-sync | ✅ |
| 3. Service Mesh & Zero-Trust (Istio) | `task-3/` | mTLS STRICT, identity-based `AuthorizationPolicy`, defense-in-depth `NetworkPolicy` | 🚧 not started |
| 4. Recon & Penetration Testing | `task-4/` | Passive OSINT attack-surface report + authorized pentest write-up | 🚧 not started |

## Quick start (local, no cloud account required)

```bash
# 1. Spin up the local 3-node cluster (1 control-plane, 2 workers)
kind create cluster --config task-1/kind-config.yml

# 2. Deploy the hardened workload
kubectl apply -f task-1/deploy/namespace.yaml
kubectl apply -f task-1/deploy/

# 3. Apply Kyverno guardrail policies (requires Kyverno installed on the cluster)
kubectl apply -f task-1/deploy/policies/

# 4. Point ArgoCD at this repo for GitOps-managed deploys
kubectl apply -f task-2/application.yml
```

> Note: `task-1/deploy/secret.yml` is a `bitnami.com/v1alpha1 SealedSecret` — it
> requires the Sealed Secrets controller installed on the cluster to decrypt
> (`kubeseal`'s controller-side key), otherwise `DB_PASSWORD`/`STRIPE_API_KEY` won't
> resolve into a real `Secret`.

## CI/CD pipeline (`devsecops.yml`)

Triggered on push to `main`/`task2-secure-cicd-supply-chain` and on PRs to `main`:

```
semgrep ─┐
gitleaks ─┼─► build-push (GHCR) ─┬─► cosign (sign + verify)  ─┐
dependency ─┤                     └─► slsa (provenance)        ┼─► gitops-cd
image-scan ─┘                                                  ┘   (commits new image
                                                                     digest to
                                                                     task-1/deploy/deployment.yaml)
```

All four scan jobs (`semgrep`, `gitleaks`, `dependency`, `trivy-image`) gate the build —
`build-push` only runs once all four pass. `cosign` and `slsa` both run against the
freshly built image's digest, and `gitops-cd` runs last, committing the new
SHA-tagged image back into `task-1/deploy/deployment.yaml` for ArgoCD to pick up.

See [`task-2/README.md`](./task-2/README.md) for the full gate-by-gate fail policy.

## Architecture

> Add an architecture diagram here (draw.io / Excalidraw export) showing `ledger-api`
> + `reporting` in the `payments` namespace, the ingress-nginx path in, and the
> CI/CD → GHCR → ArgoCD delivery loop. Once Task 3 lands, extend it with the Istio
> mesh boundary around the PCI cardholder-data-environment (CDE).

## Evaluation mapping

| Criteria | Where it's addressed |
|---|---|
| Security Hardening | `task-1/` |
| Secure Delivery | `.github/workflows/`, `task-2/` |
| Zero-Trust & Networking | `task-1/deploy/networkpolicy.yml` (ingress-only today), `task-3/` (pending) |
| Offensive Skill | `task-4/` (pending) |
| Automation & Quality | Fully automated `devsecops.yml` pipeline, GitOps auto-sync |
| Judgement | See design-decision notes in `task-1/README.md` and `task-2/README.md` |

## Known gaps / what's left

- Task 3 (Istio zero-trust) and Task 4 (recon + pentest) not started.
- `networkpolicy.yml` currently defines ingress-only rules — no egress default-deny yet.
- Bonus items not yet done: SARIF upload to the repo Security tab, Pod Security
  Standards (`restricted`) at the namespace level, RBAC personas (dev/operator/admin),
  canary/blue-green rollout.

## Notes

- Entire assessment runs locally and free — `kind` (3-node cluster) + GitHub Actions +
  GHCR, no cloud account required.
- AI assistants were used as a development aid; all artifacts and findings can be
  walked through and defended in the follow-up conversation.
