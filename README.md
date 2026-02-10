# IBM Event Automation GitOps

Production deployment of IBM Event Automation on OpenShift using ArgoCD and Sealed Secrets.

Tested on **ROKS** (IBM Cloud). See [ROSA notes](#rosa-deployment) if deploying to AWS.

## Components

- Event Streams (Kafka) — SCRAM-SHA-512 auth
- Event Processing (Flink) — OIDC auth
- Event Endpoint Management — OIDC auth
- Flink

## Structure

```
argocd-app-of-apps.yaml    # Single entry point
argocd-apps/               # ArgoCD application definitions
operators/                 # Operator subscriptions + sealed secrets
instances/                 # Component instances
docs/                      # Installation and configuration guides
```

## Quick Start (ROKS)

1. Fork this repo and update `repoURL` in `argocd-app-of-apps.yaml` and all files in `argocd-apps/`
2. Install `kubeseal` CLI: `brew install kubeseal`
3. Create sealed secrets (IBM Entitlement Key + OIDC secrets) — see [docs/INSTALL.md](docs/INSTALL.md)
4. Commit and push
5. Deploy: `oc apply -f argocd-app-of-apps.yaml`

ArgoCD sync waves handle deployment order automatically:
- Wave 0: Operators (including Sealed Secrets controller)
- Wave 1: Instances (after secrets are decrypted)

## ROSA Deployment

Before creating sealed secrets and deploying, update the following:

1. **Storage class** — change `ocs-external-storagecluster-ceph-rbd` to `gp3-csi` in instance files (look for `# ROSA:` comments)
2. **OIDC provider** — update EP and EEM instance CRs to use Azure AD issuer URL and claim pointers (see `# ROSA:` comments in instance files)
3. **Sealed secrets** — generate OIDC sealed secrets with Azure AD credentials instead of Keycloak

See [docs/ROSA-INSTALL-SEALED-SECRETS.md](docs/ROSA-INSTALL-SEALED-SECRETS.md) for the full ROSA guide.

## Documentation

- [docs/INSTALL.md](docs/INSTALL.md) — Installation instructions
- [docs/ROSA-INSTALL-SEALED-SECRETS.md](docs/ROSA-INSTALL-SEALED-SECRETS.md) — ROSA-specific guide (Azure AD OIDC)
- [docs/EP-ES-CONNECTION.md](docs/EP-ES-CONNECTION.md) — Connecting Event Processing to Event Streams

## Useful Commands

**Get UI routes:**
```bash
oc get routes -n event-streams && echo "---" && oc get routes -n event-processing && echo "---" && oc get routes -n event-endpoint-mgmt
```

**Get Kafka admin password:**
```bash
oc get secret es-prod-admin -n event-streams -o jsonpath='{.data.password}' | base64 -d && echo ""
```
