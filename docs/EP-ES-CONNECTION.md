# Connecting Event Processing to Event Streams

## Known Limitations

- **ES UI cannot generate credentials without IAM.** The ES UI "Connect to this cluster" / "Generate credentials" feature requires IBM Cloud Pak foundational services (IAM) authentication. SCRAM-authenticated users get a 403 error. All credentials must be created via gitops (KafkaUser CRs) or `oc` CLI instead.

- **Accept certificate prompt.** The EP wizard will prompt you to accept the ES TLS certificate when connecting to either listener. This is expected — tick the checkbox to proceed.

---

## Historical Note: ES 12.2.1 TLS Bug

ES 12.2.1 had a bug where brokers sent only the leaf certificate during the TLS handshake
(`tls.pemChainIncluded=false`), breaking EP and Flink connectivity. This was resolved by
running ES **12.2.0** (confirmed working). Do not upgrade to 12.2.1.

The ES subscription is pinned to `startingCSV: ibm-eventstreams.v12.2.0` with
`installPlanApproval: Manual` to prevent automatic upgrade to 12.2.1.

---

## Connection Options

Both the **Internal SCRAM** and **External SCRAM** listeners work with EP event sources. Prefer the internal listener for in-cluster deployments (lower latency, no egress).

---

## Internal SCRAM Connection (Recommended)

Use this when EP and ES are in the same cluster.

### 1. Get the Internal Bootstrap URL

In the ES UI: **Connect to this cluster → Internal**. Use the hostname shown there.

### 2. Get SCRAM Credentials

```bash
oc get secret es-prod-admin -n event-streams \
  -o jsonpath='{.data.password}' | base64 -d && echo ""
```

Username: `es-prod-admin`

### 3. Configure Event Source in EP

1. Open the EP UI and create a new flow
2. Add an **Event source** node
3. Enter the internal bootstrap URL from the ES UI
4. Accept the certificate when prompted
5. Select **SCRAM-SHA-512**, enter credentials
6. Select your topic
7. Click **Configure**

---

## External SCRAM Connection

Use this when EP is in a different cluster, or as a fallback.

### 1. Get the External Bootstrap URL

In the ES UI: **Connect to this cluster → External**. Use the hostname shown there.

### 2. Get SCRAM Credentials

```bash
oc get secret es-prod-admin -n event-streams \
  -o jsonpath='{.data.password}' | base64 -d && echo ""
```

Username: `es-prod-admin`

### 3. Configure Event Source in EP

1. Open the EP UI and create a new flow
2. Add an **Event source** node
3. Enter the external bootstrap URL from the ES UI
4. Accept the certificate when prompted
5. Select **SCRAM-SHA-512**, enter credentials
6. Select your topic
7. Click **Configure**

---

## Known Open Issues

| Issue | Impact | Status |
|---|---|---|
| ES UI Producers/Monitoring tab missing | ES metrics UI incomplete | Under investigation |
