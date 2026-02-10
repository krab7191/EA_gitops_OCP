# ROSA Installation Guide (Sealed Secrets + Keycloak)

## Prerequisites

1. **ROSA cluster** with cluster-admin access (OpenShift 4.17+)
2. **IBM Entitlement Key** from [myibm.ibm.com/products-services/containerlibrary](https://myibm.ibm.com/products-services/containerlibrary)
3. **Keycloak instance** (for Event Processing and Event Endpoint Management) with:
   - Realm created
   - Client configured (confidential)
   - Client ID and Secret
4. **CLI tools**: `oc`, `git`, `kubeseal`

## Setup

### 1. Fork/Clone Repository

```bash
git clone https://github.com/YOUR_ORG/EA_gitops_OCP.git
cd EA_gitops_OCP
```

### 2. Update Git URLs

```bash
export GIT_ORG="your-github-username"

# Update all Git URLs
find argocd-apps bootstrap -name "*.yaml" -exec sed -i.bak \
  "s|YOUR_ORG|${GIT_ORG}|g" {} \;

# Clean up and commit
find . -name "*.bak" -delete
git add .
git commit -m "Update Git URLs"
git push
```

### 3. Install Sealed Secrets Controller

```bash
# Install Sealed Secrets operator
oc apply -f bootstrap/sealed-secrets/subscription.yaml

# Wait for controller (2-3 minutes)
oc wait --for=condition=Ready pod \
  -l name=sealed-secrets-controller \
  -n sealed-secrets \
  --timeout=300s

# Install kubeseal CLI (macOS)
brew install kubeseal

# Verify
kubeseal --version
```

### 4. Create Sealed Secrets

#### IBM Entitlement Key

```bash
export IBM_ENTITLEMENT_KEY="your-entitlement-key"

# Create secret (don't apply)
oc create secret docker-registry ibm-entitlement-key \
  --docker-server=cp.icr.io \
  --docker-username=cp \
  --docker-password="${IBM_ENTITLEMENT_KEY}" \
  --docker-email=your-email@example.com \
  -n openshift-operators \
  --dry-run=client -o yaml > /tmp/ibm-entitlement-key.yaml

# Seal it
kubeseal --format=yaml \
  --controller-namespace=sealed-secrets \
  --controller-name=sealed-secrets-controller \
  < /tmp/ibm-entitlement-key.yaml \
  > operators/ibm-catalogs/sealed-entitlement-key.yaml

# Clean up
rm /tmp/ibm-entitlement-key.yaml
```

#### Keycloak OIDC Secrets (Event Processing & Event Endpoint Management)

> **Note:** Event Streams uses SCRAM-SHA-512 for Admin UI authentication — no OIDC secret needed.
> The operator auto-generates SCRAM credentials at deploy time. See [Post-Install](#post-install) for how to retrieve them.

```bash
# Set Keycloak values
export KEYCLOAK_URL="https://keycloak.example.com"
export KEYCLOAK_REALM="event-automation"
export KEYCLOAK_CLIENT_ID="event-automation"
export KEYCLOAK_CLIENT_SECRET="your-client-secret"
export DISCOVERY_URL="${KEYCLOAK_URL}/realms/${KEYCLOAK_REALM}/.well-known/openid-configuration"

# Event Processing
oc create secret generic oidc-client-secret \
  --from-literal=client-id="${KEYCLOAK_CLIENT_ID}" \
  --from-literal=client-secret="${KEYCLOAK_CLIENT_SECRET}" \
  --from-literal=discovery-url="${DISCOVERY_URL}" \
  -n event-processing \
  --dry-run=client -o yaml | \
kubeseal --format=yaml \
  --controller-namespace=sealed-secrets \
  --controller-name=sealed-secrets-controller \
  > config/sealed-secrets/event-processing-oidc-secret.yaml

# Event Endpoint Management
oc create secret generic oidc-client-secret \
  --from-literal=client-id="${KEYCLOAK_CLIENT_ID}" \
  --from-literal=client-secret="${KEYCLOAK_CLIENT_SECRET}" \
  --from-literal=discovery-url="${DISCOVERY_URL}" \
  -n event-endpoint-mgmt \
  --dry-run=client -o yaml | \
kubeseal --format=yaml \
  --controller-namespace=sealed-secrets \
  --controller-name=sealed-secrets-controller \
  > config/sealed-secrets/event-endpoint-mgmt-oidc-secret.yaml
```

### 5. Commit Sealed Secrets

```bash
# Verify sealed secrets were created
ls -la config/sealed-secrets/
ls -la operators/ibm-catalogs/sealed-entitlement-key.yaml

# Commit to Git
git add config/sealed-secrets/ operators/ibm-catalogs/sealed-entitlement-key.yaml
git commit -m "Add sealed secrets for deployment"
git push
```

## Deploy

### 1. Install ArgoCD

```bash
# Install ArgoCD operator
oc apply -f bootstrap/argocd/subscription.yaml

# Wait for ArgoCD (2-3 minutes)
oc wait --for=condition=Ready pod \
  -l name=openshift-gitops-operator \
  -n openshift-operators --timeout=300s
```

### 2. Deploy Everything

```bash
# Deploy the app-of-apps
oc apply -f bootstrap/argocd/argocd-app-of-apps.yaml
```

This will deploy in order:
1. Sealed Secrets controller (sync-wave: -1)
2. Operators (sync-wave: 0)
3. Sealed secrets (sync-wave: 1) - decrypted by controller
4. Instances (sync-wave: 2)

## Monitor

```bash
# Get ArgoCD UI
ARGOCD_URL=$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}')
ARGOCD_PASS=$(oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d)

echo "ArgoCD: https://${ARGOCD_URL}"
echo "User: admin"
echo "Pass: ${ARGOCD_PASS}"

# Watch applications
watch oc get applications -n openshift-gitops

# Check sealed secrets are decrypted
oc get secrets -n openshift-operators | grep ibm-entitlement-key
oc get secrets -n event-processing | grep oidc-client-secret
oc get secrets -n event-endpoint-mgmt | grep oidc-client-secret

# Check instances (15-30 min to be ready)
oc get eventstreams,eventprocessing,eventendpointmanagement,flinkdeployment --all-namespaces
```

## Post-Install

### Retrieve Event Streams SCRAM Credentials

Event Streams uses SCRAM-SHA-512 for Admin UI authentication. The operator auto-generates
an admin user at deploy time. Retrieve the credentials:

```bash
# Get the admin username
oc get secret production-cluster-ibm-es-ac-reg-admin -n event-streams \
  -o jsonpath='{.data.username}' | base64 -d && echo

# Get the admin password
oc get secret production-cluster-ibm-es-ac-reg-admin -n event-streams \
  -o jsonpath='{.data.password}' | base64 -d && echo

# Get the Admin UI URL
echo "https://$(oc get route production-cluster-ibm-es-ui -n event-streams -o jsonpath='{.spec.host}')"
```

Use these credentials to log into the Event Streams Admin UI.

### Add Keycloak Redirect URIs (Event Processing & Event Endpoint Management)

Get the routes and add them to your Keycloak client:

```bash
echo "Event Processing: https://$(oc get route production-processing-ibm-ep-ui -n event-processing -o jsonpath='{.spec.host}')/callback"
echo "Event Endpoint Management: https://$(oc get route production-eem-ibm-eem-manager -n event-endpoint-mgmt -o jsonpath='{.spec.host}')/callback"
```

Add these to Keycloak → Clients → event-automation → Valid Redirect URIs

## Troubleshooting

**Sealed secret not decrypting:**
```bash
# Check controller logs
oc logs -n sealed-secrets deployment/sealed-secrets-controller

# Check sealed secret status
oc describe sealedsecret <name> -n <namespace>
```

**Operators not installing:**
```bash
oc get catalogsource ibm-operator-catalog -n openshift-marketplace
oc describe subscription -n event-streams
```

**Instance not ready:**
```bash
oc describe eventstreams production-cluster -n event-streams
oc get events -n event-streams --sort-by='.lastTimestamp' | tail -20
```

## Additional Resources

- [Sealed Secrets Setup](./SEALED-SECRETS-SETUP.md) - Detailed sealed secrets guide
- [Keycloak Setup](./KEYCLOAK-SETUP.md) - Keycloak configuration details
