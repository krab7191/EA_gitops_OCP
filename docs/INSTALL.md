# Installation Guide

## Prerequisites

- ROKS/ROSA cluster with cluster-admin access
- IBM Entitlement Key from myibm.ibm.com
- Keycloak: Issuer URL, Client ID, Client Secret (for Event Processing and Event Endpoint Management)
- Tools: `oc`, `git`, `kubeseal`

Install kubeseal:
```bash
brew install kubeseal  # macOS
```

## Installation

### 1. Create Sealed Secrets

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

**Keycloak OIDC secrets (Event Processing & Event Endpoint Management):**
```bash
export KEYCLOAK_ISSUER_URL="https://keycloak.example.com/realms/your-realm"
export KEYCLOAK_CLIENT_ID="your-client-id"
export KEYCLOAK_CLIENT_SECRET="your-secret"
export DISCOVERY_URL="${KEYCLOAK_ISSUER_URL}/.well-known/openid-configuration"

# Event Processing
oc create secret generic oidc-client-secret \
  --from-literal=client-id="${KEYCLOAK_CLIENT_ID}" \
  --from-literal=client-secret="${KEYCLOAK_CLIENT_SECRET}" \
  --from-literal=discovery-url="${DISCOVERY_URL}" \
  -n event-processing --dry-run=client -o yaml | \
kubeseal --format=yaml --controller-namespace=sealed-secrets \
  > operators/ibm-catalogs/event-processing-oidc-secret.yaml

# Event Endpoint Management
oc create secret generic oidc-client-secret \
  --from-literal=client-id="${KEYCLOAK_CLIENT_ID}" \
  --from-literal=client-secret="${KEYCLOAK_CLIENT_SECRET}" \
  --from-literal=discovery-url="${DISCOVERY_URL}" \
  -n event-endpoint-mgmt --dry-run=client -o yaml | \
kubeseal --format=yaml --controller-namespace=sealed-secrets \
  > operators/ibm-catalogs/event-endpoint-mgmt-oidc-secret.yaml
```

> **Note:** Event Streams uses SCRAM-SHA-512 authentication for the Admin UI (not OIDC).
> The operator automatically generates admin credentials at deployment time.
> No sealed secret is needed — see [Retrieving Event Streams SCRAM credentials](#5-access) below.

### 2. Commit and Push

```bash
git add operators/ibm-catalogs/
git commit -m "Add sealed secrets"
git push
```

### 3. Deploy

```bash
oc apply -f argocd-app-of-apps.yaml
```

Wait 15-30 minutes for everything to deploy.

### 4. Monitor

```bash
oc get applications -n openshift-gitops
oc get pods --all-namespaces | grep -E "event-streams|event-processing|event-endpoint|flink"
```

### 5. Access

ArgoCD UI:
```bash
echo "https://$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}')"
oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d
```

Event Streams Admin UI (SCRAM credentials):
```bash
# The operator auto-generates an admin user. Retrieve the credentials:
oc get secret production-cluster-ibm-es-ac-reg-admin -n event-streams -o jsonpath='{.data.username}' | base64 -d
oc get secret production-cluster-ibm-es-ac-reg-admin -n event-streams -o jsonpath='{.data.password}' | base64 -d

# Get the Admin UI URL:
echo "https://$(oc get route production-cluster-ibm-es-ui -n event-streams -o jsonpath='{.spec.host}')"
```

Event Processing & Event Endpoint Management UIs (add these to Keycloak redirect URIs):
```bash
oc get routes -n event-processing -n event-endpoint-mgmt
```