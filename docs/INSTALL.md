# Installation Guide

## Prerequisites

- ROKS/ROSA cluster with cluster-admin access
- IBM Entitlement Key from myibm.ibm.com
- OIDC Provider (Keycloak or Azure AD): Issuer URL, Client ID, Client Secret
- Tools: `oc`, `git`, `kubeseal`

Install kubeseal:
```bash
brew install kubeseal  # macOS
```

## Installation

### 1. Fork and Clone

Fork this repo, then update the `repoURL` in these files to point to your fork:
- `argocd-app-of-apps.yaml`
- All files in `argocd-apps/`

Look for the `# ROSA:` comments in the instance files if deploying to ROSA (storage class, OIDC config).

### 2. Create Sealed Secrets

All sealed secrets go in `operators/ibm-catalogs/` directory.

**IBM Entitlement Key:**
```bash
export IBM_ENTITLEMENT_KEY="your-key"

oc create secret docker-registry ibm-entitlement-key \
  --docker-server=cp.icr.io --docker-username=cp \
  --docker-password="${IBM_ENTITLEMENT_KEY}" \
  -n openshift-operators --dry-run=client -o yaml | \
kubeseal --format=yaml --controller-namespace=sealed-secrets \
  > operators/ibm-catalogs/sealed-entitlement-key.yaml
```

**OIDC secrets (Event Processing & Event Endpoint Management):**
```bash
# For Keycloak:
export OIDC_ISSUER_URL="https://keycloak.example.com/realms/your-realm"
# For Azure AD:
# export OIDC_ISSUER_URL="https://login.microsoftonline.com/<tenant-id>/v2.0"

export OIDC_CLIENT_ID="your-client-id"
export OIDC_CLIENT_SECRET="your-secret"
export DISCOVERY_URL="${OIDC_ISSUER_URL}/.well-known/openid-configuration"

# Event Processing
oc create secret generic oidc-client-secret \
  --from-literal=client-id="${OIDC_CLIENT_ID}" \
  --from-literal=client-secret="${OIDC_CLIENT_SECRET}" \
  --from-literal=discovery-url="${DISCOVERY_URL}" \
  -n event-processing --dry-run=client -o yaml | \
kubeseal --format=yaml --controller-namespace=sealed-secrets \
  > operators/ibm-catalogs/event-processing-oidc-secret.yaml

# Event Endpoint Management
oc create secret generic oidc-client-secret \
  --from-literal=client-id="${OIDC_CLIENT_ID}" \
  --from-literal=client-secret="${OIDC_CLIENT_SECRET}" \
  --from-literal=discovery-url="${DISCOVERY_URL}" \
  -n event-endpoint-mgmt --dry-run=client -o yaml | \
kubeseal --format=yaml --controller-namespace=sealed-secrets \
  > operators/ibm-catalogs/event-endpoint-mgmt-oidc-secret.yaml
```

> **Note:** Event Streams uses SCRAM-SHA-512 authentication for the Admin UI (not OIDC).
> The operator generates credentials from the KafkaUser CR (`admin-user.yaml`).
> No sealed secret is needed — see [Retrieving Event Streams SCRAM credentials](#5-access) below.

### 3. Configure OIDC Provider

In your OIDC provider (Keycloak or Azure AD), configure the client:

**Valid Redirect URIs** (add after deployment when you know the routes, or use wildcards):
```
https://<ep-route>/*
https://<eem-route>/*
```

**Client Roles** — create these roles on the OIDC client:

For Event Processing (only one role):
- `user` — full access to EP UI

For Event Endpoint Management:
- `admin` — manage gateways and certificates
- `author` — create and share resources
- `viewer` — read-only access

Assign appropriate roles to your users.

**Token Mapper** — ensure roles appear in the token:
- Add a "User Client Role" mapper
- Set Token Claim Name to `roles`
- Enable for both ID token and access token

### 4. Commit and Push

```bash
git add operators/ibm-catalogs/
git commit -m "Add sealed secrets"
git push
```

### 5. Deploy

```bash
oc apply -f argocd-app-of-apps.yaml
```

Wait 15-30 minutes for everything to deploy.

### 6. Monitor

```bash
oc get applications -n openshift-gitops
oc get pods -n event-streams
oc get pods -n event-processing
oc get pods -n event-endpoint-mgmt
oc get pods -n flink
```

### 7. Access

**ArgoCD UI:**
```bash
echo "https://$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}')"
oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d
```

**Event Streams Admin UI (SCRAM credentials):**
```bash
# Username is es-prod-admin. Retrieve the password:
oc get secret es-prod-admin -n event-streams -o jsonpath='{.data.password}' | base64 -d

# Get the Admin UI URL:
echo "https://$(oc get route es-prod-ibm-es-ui -n event-streams -o jsonpath='{.spec.host}')"
```

**Event Processing UI (OIDC login):**
```bash
echo "https://$(oc get route production-processing-ibm-ep-rt -n event-processing -o jsonpath='{.spec.host}')"
```

**Event Endpoint Management UI (OIDC login):**
```bash
echo "https://$(oc get route production-eem-ibm-eem-manager -n event-endpoint-mgmt -o jsonpath='{.spec.host}')"
```

> After getting the EP and EEP routes, add them as Valid Redirect URIs in your OIDC provider if you haven't already.
