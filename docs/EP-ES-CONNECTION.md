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

After configuring an event source or sink in the EP wizard, the generated SQL will contain
`ssl.truststore.certificates` set to the bootstrap broker's leaf cert. This must be manually
replaced with the cluster CA cert so EP's Kafka client can validate all per-broker route certs.

### 1. Get the cluster CA cert

```bash
oc get secret es-prod-cluster-ca-cert -n event-streams \
  -o jsonpath='{.data.ca\.crt}' | base64 -d
```

Copy the full PEM output (`-----BEGIN CERTIFICATE-----` through `-----END CERTIFICATE-----`).

### 2. Edit the event source SQL

In the EP flow canvas, click the event source node → **Edit** → **Preview SQL**.

Find the `ssl.truststore.certificates` property and replace the leaf cert with the CA cert:

```sql
'properties.ssl.truststore.certificates' = '-----BEGIN CERTIFICATE-----
<paste cluster CA cert here>
-----END CERTIFICATE-----
',
```

Submit the updated SQL.

### 3. Repeat for every sink

EP auto-fills the leaf cert for every new event source and sink. Repeat step 2 for each
sink node's SQL.

### Notes

- This replacement must be done manually each time an event source or sink is created.
  There is no GitOps-friendly way to pre-configure this — it is stored in EP's persistent
  storage, not in the CR.
- The `trustedCertificates` field in the EP CR is still useful: it allows Vert.x to trust
  the bootstrap URL without a cert acceptance prompt, which prevents session timeouts during
  wizard navigation. Keep it configured.
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
| EP UI flow preview blank | Cannot see events on flow canvas | Check sink topic in ES UI |
| ES UI Producers/Monitoring "Uh oh" error | ES metrics UI broken | Under investigation |
