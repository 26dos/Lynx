
# 🌐 Filecoin Global Retriever Deployment Plan (English Version)

## 1. Project Objective

### 🌍 Purpose of Global Deployment

- Accurately evaluate SP service capability, latency, and accessibility worldwide;
- Eliminate regional network bottleneck bias in retrieval results;
- Better distinguish SPs providing real data from those injecting garbage/fillers;
- Lay the foundation for trust scores, incentive penalties, and subsidy mechanisms.

---

## 2. Deployment Regions (7 Continents Coverage)

| Continent | Suggested Region     | Cloud Providers          |
|-----------|----------------------|--------------------------|
| Asia      | Singapore / Tokyo    | AWS, Alibaba, local SG   |
| Europe    | Frankfurt / Netherlands | Hetzner, AWS         |
| North America | Virginia (USA)   | AWS, DigitalOcean, Vultr |
| South America | São Paulo (Brazil) | AWS, Google Cloud      |
| Africa    | Johannesburg         | Azure, local VPS         |
| Oceania   | Sydney (Australia)   | AWS, GCP                 |
| Antarctica| Simulated region via southern proxy | Not applicable |

> You may select 6 regions for deployment, simulating Antarctica via proxy if needed.

---

## 3. System Architecture Overview

```
                                ┌─────────────────────────┐
                                │     Central Control     │
                                │    (Coordinator/API)     │
                                └──────────┬──────────────┘
                                           │
        ┌──────────────────────────────────┼─────────────────────────────────┐
        ▼                                  ▼                                 ▼
┌────────────┐                   ┌────────────────┐               ┌────────────────┐
│ Asia Node   │                   │ Europe Node     │               │ America Node        │
│ (Singapore) │                   │ (Frankfurt)     │               │ (Virginia)     │
└────┬───────┘                   └────┬────────────┘               └────┬────────────┘
     │                                │                                 │
     ▼                                ▼                                 ▼
┌────────────┐               ┌────────────────┐               ┌────────────────┐
│ Retriever   │               │ Retriever       │               │ Retriever       │
│ IP Check    │               │ Latency Check   │               │ Verification    │
└────┬───────┘               └────┬────────────┘               └────┬────────────┘
     │                                │                                 │
     ▼                                ▼                                 ▼
      Push data to MongoDB + Prometheus for unified storage and visualization
```

---

## 4. System Components

### 1. Control Center (Main Node)

- Dispatch tasks to all regional agents;
- Launch distributed jobs (via gRPC or REST);
- Aggregate results into dashboard;
- Automatically compare performance across regions.

---

### 2. Regional Retrieval Node Components

#### a. Retriever Agent
- Sends retrieval requests to SPs;
- Logs response status, latency, size;
- Records SP IP and PeerID.

#### b. Content Verifier
- Verifies if returned CAR contains expected CID;
- Performs entropy/magic-byte analysis;
- Confirms data validity and completeness.

#### c. Ping/Geo/IP Module
- Measures P2P and HTTP latency to SP;
- Geolocates IP, detects VPN/hosting.

#### d. Local Cache Module
- Caches results locally for 24 hours;
- Uploads asynchronously to main MongoDB.

---

## 5. Data Storage & Visualization

### MongoDB Document Structure (Simplified)

```json
{
  "timestamp": "2025-06-11T13:12:00Z",
  "region": "Europe",
  "retriever_id": "eu-node-01",
  "target_sp": "f012345",
  "ip": "45.12.x.x",
  "latency_ms": 820,
  "retrieval_success": true,
  "data_verified": true,
  "risk_flags": ["high_entropy", "no_magic"],
  "car_size_bytes": 2048000
}
```

### Grafana + Prometheus Dashboards

- 🌍 Retrieval success rate by continent (heatmap);
- 📉 SP response latency distribution;
- 🧠 Failure clustering (VPN, missing data, etc.);
- 🗺️ Geo-distribution of SPs vs actual availability.

---

## 6. Recommended Technologies

| Function       | Tools/Tech                             |
|----------------|----------------------------------------|
| Multi-region Job Dispatch | Golang cron / Redis + worker pool |
| Communication   | gRPC (preferred for bi-directional)    |
| Deployment      | Docker + SystemD / Kubernetes          |
| Monitoring/Storage | MongoDB + Grafana (export or push)  |
| Network Analysis | libp2p, HTTP, ipinfo.io, IPQS, geoip2 |

---

## 7. Security and Operations

- Each retriever only exposes secure gRPC endpoints;
- Control center can restart nodes, monitor liveness;
- All data transferred via TLS (e.g., gRPC over TLS);
- Logs routed through Loki (Prometheus stack).

---

## 8. Future Enhancements

- ⚠️ Real-time Alerts: Detect downtime or sudden latency spikes;
- 🏆 Trust Score System: Add region-based and content-based ratings for SPs;
- 🔗 IPNI Integration: Auto-verify advertised CIDs are truly retrievable;
- 🔍 Filecoin Plus Integration: Flag suspicious SPs for further auditing.
