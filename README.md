# IBM Event Automation GitOps

Production deployment of IBM Event Automation on ROKS/ROSA with Sealed Secrets. Event Streams uses SCRAM-SHA-512 auth; Event Processing and Event Endpoint Management use Keycloak OIDC.

## Components

- Event Streams (Kafka)
- Event Processing (Flink)
- Event Endpoint Management
- Flink

## Structure

```
argocd-app-of-apps.yaml    # Single entry point
argocd-apps/               # ArgoCD application definitions
operators/                 # Operator subscriptions + sealed secrets
instances/                 # Component instances
```

## Quick Start

1. Install kubeseal CLI
2. Create sealed secrets (see [docs/INSTALL.md](docs/INSTALL.md))
3. Commit and push
4. Deploy: `oc apply -f argocd-app-of-apps.yaml`

## Deployment Order

ArgoCD sync waves handle deployment order automatically:
- Wave 0: Operators (including Sealed Secrets controller)
- Wave 1: Instances (after secrets are decrypted)

## Documentation

See [docs/INSTALL.md](docs/INSTALL.md) for complete installation instructions.

## Useful commands: 

### Get UI Routes: 
`oc get routes -n event-streams && echo "---" && oc get routes -n event-processing && echo "---" && oc get routes -n event-endpoint-mgmt`

### Get kafka admin password
`oc get secret es-prod-admin -n event-streams -o jsonpath='{.data.password}' | base64 -d`

## Troubleshooting

```bash
# Check applications
oc get applications -n openshift-gitops

# Check instances
oc get eventstreams,eventprocessing,eventendpointmanagement,flinkdeployment --all-namespaces

# Check pods
oc get pods --all-namespaces | grep -E "event-streams|event-processing|event-endpoint|flink"
```

