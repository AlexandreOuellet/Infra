# base/ — Kubernetes Manifests

## OVERVIEW

Kustomize base for cluster-level infrastructure components (not application workloads — those live in `arr` and `ai`). Each component is vendored/adapted from its upstream install manifests rather than following the 4-file app pattern used elsewhere.

## COMPONENTS (5)

| Component | Purpose | Notes |
|-----------|---------|-------|
| longhorn | Distributed block storage (StorageClasses, CSI driver) | Vendored install manifest (`00-longhorn.yaml`), single-node cluster so replicas are tuned down from upstream defaults — see CONVENTIONS |
| metallb | Bare-metal LoadBalancer implementation | Provides external IPs for Services/Ingress on-prem |
| cert-manager | TLS certificate automation | `letsencrypt-prod` ClusterIssuer used by app ingresses across `arr`/`ai` |
| ArgoCD | GitOps continuous delivery | Points at the overlay repo for this environment (see root `README.md`) |
| kubeseal | Sealed Secrets controller | Lets `arr`/`ai`/etc. commit `02-sealed-secret.yaml` files safely; seal with the matching `--namespace` |

## PLANNED

- **Authentik** — OIDC identity provider, not yet added. Not about ingress-level auth (ingresses stay open, matching current cluster convention) — the goal is per-app login: Authentik federates with Google/GitHub as upstream social providers, and individual apps that support OIDC (e.g. jellyseerr, openwebui) are configured to use Authentik as their OIDC issuer, giving users "login with Gmail/GitHub" on those apps. When adding: follow the vendored-manifest pattern (own subdirectory, `00-`/`01-` prefixed files).

## CONVENTIONS

- Component name: subdirectory matching the upstream project name (lowercase)
- File prefixes: `00-` for the core install/CRDs, `01-`/`02-`/... for ingress, extra StorageClasses, ClusterIssuers, etc. — numeric order reflects apply/dependency order
- This is a **single-node cluster**: storage and node-affinity settings are intentionally reduced from upstream multi-node defaults (e.g. Longhorn `numberOfReplicas: 1` — extra replicas would just sit on the same disk, no real redundancy benefit, but see the tradeoff noted in `longhorn/00-longhorn.yaml` and `longhorn/02-longhorn-hdd-storageclass.yaml`: single-replica volumes have no failover, so a transient node/network blip can fault a volume until manually recovered)
- Two StorageClasses exist: default (SSD) and `longhorn-hdd` (ZFS-backed HDD pool on the same Proxmox host) — `longhorn-hdd` is the sole default (`is-default-class: "true"`); IO-sensitive workloads (e.g. jellyfin-cache) should stay off it deliberately

## ANTI-PATTERNS

- DO NOT hand-edit vendored upstream manifests beyond the specific overrides already commented in them (e.g. longhorn's storageclass ConfigMap section) — makes future upstream version bumps harder to diff
- DO NOT mark more than one StorageClass as `is-default-class: "true"` — caused ambiguous default-provisioning behavior previously
- DO NOT bump replica counts back to multi-node defaults without confirming this is still a single-node cluster

## EDITING

```bash
# Add new infra component
mkdir base/newcomponent/
# Vendor or author 00-<component>.yaml (+ 01-ingress.yaml etc. as needed)
# Add to base/kustomization.yaml
```
