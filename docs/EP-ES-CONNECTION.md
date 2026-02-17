# Connecting Event Processing to Event Streams

## Known Limitations

- **ES UI cannot generate credentials without IAM.** The ES UI "Connect to this cluster" /
  "Generate credentials" feature requires IBM Cloud Pak foundational services (IAM)
  authentication. SCRAM-authenticated users get a 403 error. All credentials must be created
  via gitops (KafkaUser CRs) or `oc` CLI instead.

## Connection Options

Event Processing supports both internal and external SCRAM-SHA-512 connections:

1. **Internal SCRAM (Recommended)** - Keeps traffic within cluster
2. **External SCRAM** - For external access or testing

---

## Option 1: Internal SCRAM Connection (Recommended)

### 1. Get the Internal Bootstrap URL

```bash
oc get eventstreams es-prod -n event-streams \
  -o jsonpath='{.status.kafkaListeners[?(@.name=="intscram")].bootstrapServers}' && echo ""
```

Expected output:
```
es-prod-kafka-bootstrap.event-streams.svc:9093
```

### 2. Get SCRAM Credentials

The `es-prod-admin` KafkaUser is provisioned via gitops (`instances/event-streams/admin-user.yaml`).
The operator auto-generates the password.

```bash
# Username: es-prod-admin
# Password:
oc get secret es-prod-admin -n event-streams \
  -o jsonpath='{.data.password}' | base64 -d && echo ""
```

### 3. Configure Event Source in EP

1. Open the Event Processing UI
2. Create a new flow (or edit existing)
3. Add an **Event source** node
4. Enter the **internal** bootstrap URL: `es-prod-kafka-bootstrap.event-streams.svc:9093`
5. Click **Accept certificates** when prompted to trust the cluster CA certificate
6. Click **Next** to proceed to **Access credentials**
7. Select **SCRAM-SHA-512** as the security mechanism
8. Enter username `es-prod-admin` and the password from step 2
9. Select your topic and continue configuring the flow

**Benefits:**
- Traffic stays within the cluster (no external route)
- Better performance (no route overhead)

---

## Option 2: External SCRAM Connection

Use this option for external access or testing from outside the cluster.

### 1. Get the External Bootstrap URL

```bash
oc get eventstreams es-prod -n event-streams \
  -o jsonpath='{.status.kafkaListeners[?(@.name=="extscram")].bootstrapServers}' && echo ""
```

Expected output:
```
es-prod-kafka-extscram-bootstrap-event-streams.apps.<cluster-domain>:443
```

### 2-3. Follow Same Steps as Internal

Use the same SCRAM credentials from Option 1, but with the external bootstrap URL in step 4.