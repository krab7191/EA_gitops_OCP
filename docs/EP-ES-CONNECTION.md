# Connecting Event Processing to Event Streams

## Known Limitations

- **ES UI cannot generate credentials without IAM.** The ES UI "Connect to this cluster" /
  "Generate credentials" feature requires IBM Cloud Pak foundational services (IAM)
  authentication. SCRAM-authenticated users get a 403 error. All credentials must be created
  via gitops (KafkaUser CRs) or `oc` CLI instead.

- **Internal SCRAM (9093) not usable for EP event sources.** The EP event source wizard
  validates the bootstrap URL before accepting credentials. The internal SCRAM listener
  returns "Connection credential not required" at the introspection step. Use the external
  SCRAM listener (port 443) instead.

---

## Historical Note: ES 12.2.1 TLS Bug

ES 12.2.1 had a bug where brokers sent only the leaf certificate during the TLS handshake
(`tls.pemChainIncluded=false`), breaking EP and Flink connectivity. This was resolved by
running ES **12.2.0** (confirmed working). Do not upgrade to 12.2.1.

The ES subscription is pinned to `startingCSV: ibm-eventstreams.v12.2.0` with
`installPlanApproval: Manual` to prevent automatic upgrade to 12.2.1.

---

## Connection Options

Only the **External SCRAM** listener works with EP event sources.

---

## External SCRAM Connection

### 1. Get the External Bootstrap URL

Use the Route hostname directly — not the ES UI "Connect to this cluster" page:

```bash
oc get route es-prod-kafka-extscram-bootstrap -n event-streams -o jsonpath='{.spec.host}'
```

Use port **443** (OpenShift Routes always expose 443 externally):

```
es-prod-kafka-extscram-bootstrap-event-streams.apps.<cluster-domain>:443
```

### 2. Get SCRAM Credentials

```bash
oc get secret es-prod-admin -n event-streams \
  -o jsonpath='{.data.password}' | base64 -d && echo ""
```

Username: `es-prod-admin`

### 3. Configure Event Source in EP

1. Open the EP UI and create a new flow
2. Add an **Event source** node
3. Enter the external bootstrap URL with port 443
4. Select **SCRAM-SHA-512**, enter credentials
5. Select your topic
6. Click **Configure** — no SQL editing required

### 4. Configure Sink (if writing back to ES)

1. Add an **Event destination** node
2. Complete the wizard with the same bootstrap URL and credentials
3. No SQL editing required

---

## Known Open Issues

| Issue | Impact | Status |
|---|---|---|
| Internal SCRAM URL rejected by EP wizard | Must use external SCRAM (port 443) | Under investigation |
| ES UI Producers/Monitoring tab missing | ES metrics UI incomplete | Under investigation |
