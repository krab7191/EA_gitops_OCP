# Connecting Event Processing to Event Streams

## Known Limitations

- **ES UI cannot generate credentials without IAM.** The ES UI "Connect to this cluster" /
  "Generate credentials" feature requires IBM Cloud Pak foundational services (IAM)
  authentication. SCRAM-authenticated users get a 403 error. All credentials must be created
  via gitops (KafkaUser CRs) or `oc` CLI instead.

- **ES 12.2.x TLS bug — manual cert replacement required.** This is a bug in **Event Streams
  12.2.x** (not in EP or Flink). ES 12.2.x brokers send only the leaf certificate during the
  TLS handshake (`tls.pemChainIncluded=false`), not the full chain. EP's Kafka client
  (behaving correctly) stores whichever cert it receives from the bootstrap broker in
  `ssl.truststore.type=PEM`. When the Kafka client then connects to individual broker routes
  (each with a different leaf cert), PKIX validation fails because the CA cert was never
  provided. The JKS truststore and `trustedCertificates` in the EP CR do not fix this — they
  only affect EP's Vert.x SSL context, not the Kafka client.
  See [workaround below](#es-122x-tls-workaround).

- **Internal SCRAM (9093) not usable for EP event sources.** The EP event source wizard
  validates the bootstrap URL before accepting credentials. The internal SCRAM listener
  returns "Connection credential not required" at the introspection step.

- **EP UI flow preview may not show events.** The CollectSink websocket (Flink → EP, port
  8887) may be blocked by network policies between the `flink` and `event-processing`
  namespaces. The Flink job processes events correctly — verify by checking the sink topic
  in the ES UI instead of relying on the EP flow canvas preview.

---

## ES 12.2.x TLS Workaround

Two approaches work. **Option 2 (JKS) is preferred** — it requires no cert content in the
SQL and is consistent across all flows.

---

### Option 2 (Preferred): JKS Truststore — remove SSL properties from SQL

The `es-ca-sync` ArgoCD PostSync job automatically builds a JKS truststore containing the
ES cluster CA and mounts it into the EP pod via the `ssl-truststore` secret. The EP CR
configures `JAVA_TOOL_OPTIONS` to point the JVM at this truststore.

When `ssl.truststore.type` and `ssl.truststore.certificates` are absent from the Flink SQL,
the Kafka client falls back to the JVM truststore, which already trusts all ES broker certs.

**After completing the event source/sink wizard**, click the node → **Edit** → **Preview SQL**
and remove these two properties:

```sql
'properties.ssl.truststore.type' = 'PEM',
'properties.ssl.truststore.certificates' = '...',
```

Leave all other properties unchanged. Do this for every event source and every sink node.

The EP UI flow canvas preview will show live events when using this approach.

---

### Option 1 (Alternative): Replace leaf cert with CA cert in SQL

If the JKS is not available, replace `ssl.truststore.certificates` with the cluster CA cert
instead of the auto-filled leaf cert:

```bash
oc get secret es-prod-cluster-ca-cert -n event-streams \
  -o jsonpath='{.data.ca\.crt}' | base64 -d
```

In **Preview SQL**, replace the cert value:

```sql
'properties.ssl.truststore.certificates' = '-----BEGIN CERTIFICATE-----
<paste cluster CA cert here>
-----END CERTIFICATE-----
',
```

Keep `'properties.ssl.truststore.type' = 'PEM'`. Repeat for every source and sink.

---

### Notes

- Both workarounds require manual SQL editing per event source and sink after the wizard
  completes. There is no GitOps-friendly way to pre-configure this.
- The `trustedCertificates` field in the EP CR is still required: it allows Vert.x to trust
  the bootstrap URL during wizard navigation without a cert acceptance prompt.
- The root fix is IBM correcting ES 12.2.x to send the full cert chain. Track the IBM
  support case for a proper patch.

---

## Connection Options

Only the **External SCRAM** listener works with EP event sources in ES 12.2.x.

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
6. Click **Preview SQL** and replace `ssl.truststore.certificates` with the CA cert (see above)

### 4. Configure Sink (if writing back to ES)

1. Add an **Event destination** node
2. Complete the wizard with the same bootstrap URL and credentials
3. Click **Preview SQL** and replace `ssl.truststore.certificates` with the CA cert

---

## Known Open Issues

| Issue | Impact | Status |
|---|---|---|
| ES 12.2.x bug: leaf-cert-only TLS | Manual cert replacement per event source/sink | Workaround above; IBM support case open |
| EP UI flow preview blank | Cannot see events on flow canvas | Resolved when using Option 2 (JKS) |
| ES UI Producers/Monitoring "Uh oh" error | ES metrics UI broken | Under investigation |
