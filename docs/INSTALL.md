# Installation Guide

These instructions are for **ROKS** (IBM Cloud) with Keycloak OIDC. For ROSA (AWS) with Azure AD, see [ROSA-INSTALL-SEALED-SECRETS.md](./ROSA-INSTALL-SEALED-SECRETS.md).

## Prerequisites

- ROKS cluster with cluster-admin access
- IBM Entitlement Key from myibm.ibm.com
- Keycloak instance with a realm, confidential client, Client ID, and Client Secret
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

### 2. Create Sealed Secrets

All sealed secrets must be created with `kubeseal` and committed to git. They are
namespace-scoped — a secret sealed for one namespace cannot be used in another.

#### IBM Entitlement Key

Required in every namespace that pulls IBM container images.

```bash
export IBM_ENTITLEMENT_KEY="your-key"

for NS in openshift-operators event-streams event-processing event-endpoint-mgmt flink; do
  oc create secret docker-registry ibm-entitlement-key \
    --docker-server=cp.icr.io --docker-username=cp \
    --docker-password="${IBM_ENTITLEMENT_KEY}" \
    -n $NS --dry-run=client -o yaml | \
  kubeseal --format=yaml --controller-namespace=sealed-secrets \
    --controller-name=sealed-secrets-controller \
    > operators/ibm-catalogs/sealed-entitlement-key-${NS}.yaml
done
```

> The file committed in `operators/ibm-catalogs/sealed-entitlement-key.yaml` covers
> `openshift-operators` only. Rename or regenerate as needed for other namespaces.

#### OIDC Secrets (Event Processing & Event Endpoint Management)

```bash
export KEYCLOAK_URL="https://keycloak.example.com"
export KEYCLOAK_REALM="your-realm"
export OIDC_CLIENT_ID="your-client-id"
export OIDC_CLIENT_SECRET="your-client-secret"
export DISCOVERY_URL="${KEYCLOAK_URL}/realms/${KEYCLOAK_REALM}/.well-known/openid-configuration"

# Event Processing
oc create secret generic oidc-client-secret \
  --from-literal=client-id="${OIDC_CLIENT_ID}" \
  --from-literal=client-secret="${OIDC_CLIENT_SECRET}" \
  --from-literal=discovery-url="${DISCOVERY_URL}" \
  -n event-processing --dry-run=client -o yaml | \
kubeseal --format=yaml --controller-namespace=sealed-secrets \
  --controller-name=sealed-secrets-controller \
  > operators/ibm-catalogs/event-processing-oidc-secret.yaml

# Event Endpoint Management
oc create secret generic oidc-client-secret \
  --from-literal=client-id="${OIDC_CLIENT_ID}" \
  --from-literal=client-secret="${OIDC_CLIENT_SECRET}" \
  --from-literal=discovery-url="${DISCOVERY_URL}" \
  -n event-endpoint-mgmt --dry-run=client -o yaml | \
kubeseal --format=yaml --controller-namespace=sealed-secrets \
  --controller-name=sealed-secrets-controller \
  > operators/ibm-catalogs/event-endpoint-mgmt-oidc-secret.yaml
```

> **Event Streams** uses SCRAM-SHA-512 for the Admin UI — no OIDC secret needed.
> Credentials are auto-generated from the `KafkaUser` CR. See [Access](#6-access) below.

#### Summary of Required Sealed Secrets

| Secret name | Namespace(s) | File location |
|---|---|---|
| `ibm-entitlement-key` | `openshift-operators`, `event-streams`, `event-processing`, `event-endpoint-mgmt`, `flink` | `operators/ibm-catalogs/` |
| `oidc-client-secret` | `event-processing`, `event-endpoint-mgmt` | `operators/ibm-catalogs/` |

### 3. Configure Keycloak

In your Keycloak realm, configure the client:

**Client Roles** — create these roles on the OIDC client:

For Event Processing (only one role):
- `user` — full access to EP UI

For Event Endpoint Management:
- `admin` — manage gateways and certificates
- `author` — create and share resources
- `viewer` — read-only access

Assign appropriate roles to your users.

**Token Mapper** — ensure roles appear in the token:
- Add a "User Client Role" mapper (Mappers tab on the client)
- Set Token Claim Name to `roles`
- Enable for both ID token and access token

**Redirect URIs** — add after deployment when you know the routes:
```
https://<ep-route>/callback
https://<eem-route>/callback
```

### 4. Deploy

```bash
oc apply -f argocd-app-of-apps.yaml
```

Wait 15-30 minutes for everything to deploy.

### 5. Monitor

```bash
oc get applications -n openshift-gitops
oc get pods -n event-streams
oc get pods -n event-processing
oc get pods -n event-endpoint-mgmt
oc get pods -n flink
```

### 6. Access

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

> After getting the EP and EEM routes, add them as Valid Redirect URIs in Keycloak (Clients → your client → Valid Redirect URIs) if you haven't already.
