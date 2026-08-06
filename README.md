# Bug Farm GitOps Infrastructure (`bug-farm.xyz`)

Declarative infrastructure management for `bug-farm.xyz` platform powered by **Talos Linux**, **Kubernetes**, and **ArgoCD**.

## Repository Structure

- `bootstrap/` — Initial setup components and Root ArgoCD Application.
- `apps/` — ArgoCD `Application` definitions (grouped by system, security, monitoring).
- `infrastructure/` — Kubernetes manifests, Helm values, and Kustomize overlays.
