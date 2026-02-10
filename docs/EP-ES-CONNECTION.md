# Connecting Event Processing to Event Streams

## Known Limitations

- **ES UI cannot generate credentials without IAM.** The ES UI "Connect to this cluster" /
  "Generate credentials" feature requires IBM Cloud Pak foundational services (IAM)
  authentication. SCRAM-authenticated users get a 403 error. All credentials must be created
  via gitops (KafkaUser CRs) or `oc` CLI instead.
- **EP event source wizard validates URL before allowing cert upload.** The wizard attempts to
  connect to the Kafka bootstrap server immediately when you enter the URL, before you can
  configure certificates or authentication. This means internal mTLS listeners (port 9093)
  fail because EP has no client cert to present. Use the external SCRAM listener instead.

## Connection Steps

All credentials and certificates are retrieved via CLI — nothing is done through the ES UI.

### 1. Get the External Bootstrap URL

```bash
oc get eventstreams es-prod -n event-streams \
  -o jsonpath='{.status.kafkaListeners[?(@.name=="external")].bootstrapServers}' && echo ""
```

Expected output:
```
es-prod-kafka-bootstrap-event-streams.apps.<cluster-domain>:443
```

### 2. Get the Cluster CA Certificate

```bash
oc get secret es-prod-cluster-ca-cert -n event-streams \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > /tmp/ca.crt
```

### 3. Get SCRAM Credentials

The `es-prod-admin` KafkaUser is provisioned via gitops (`instances/event-streams/admin-user.yaml`).
The operator auto-generates the password.

```bash
# Username: es-prod-admin
# Password:
oc get secret es-prod-admin -n event-streams \
  -o jsonpath='{.data.password}' | base64 -d && echo ""
```

### 4. Configure Event Source in EP

1. Open the Event Processing UI
2. Create a new flow (or edit existing)
3. Add an **Event source** node
4. Enter the **external** bootstrap URL from step 1
5. Click **Next** to proceed to **Access credentials**
6. Select **SCRAM-SHA-512** as the security mechanism
7. Enter username `es-prod-admin` and the password from step 3
8. Upload the CA certificate (`/tmp/ca.crt`) if prompted
9. Select your topic and continue configuring the flow

## Internal mTLS Certificates (Reference)

A dedicated TLS KafkaUser (`es-prod-ep-user`) is provisioned via gitops
(`instances/event-streams/ep-kafka-user.yaml`). These can be used if a future EP version
allows certificate configuration before URL validation, or for non-UI Kafka clients.

Retrieve the client certificates:
```bash
oc get secret es-prod-ep-user -n event-streams \
  -o jsonpath='{.data.user\.crt}' | base64 -d > /tmp/user.crt
oc get secret es-prod-ep-user -n event-streams \
  -o jsonpath='{.data.user\.key}' | base64 -d > /tmp/user.key
oc get secret es-prod-cluster-ca-cert -n event-streams \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > /tmp/ca.crt
cat /tmp/user.key /tmp/user.crt > /tmp/keystore.pem
```

To use with the internal listener:
- **Server:** `es-prod-kafka-bootstrap.event-streams.svc:9093`
- **Auth:** Mutual TLS
- **Server cert:** `ca.crt`
- **Client cert:** `keystore.pem`
