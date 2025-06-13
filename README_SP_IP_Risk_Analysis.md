# Public IP Risk Analysis for Storage Providers

## 1. Objectives and Use Cases

### 🎯 Objective

Identify whether the public IP of a storage provider (SP) belongs to a VPN, proxy, cloud service, or TOR network. This helps evaluate service reliability and traceability, especially for:

- Data retrieval trust evaluation
- Detecting potential fraud/speculation
- Risk control systems (e.g., filtering frequently changing IP nodes)
- Incentive system design (subsidies, penalties, etc.)

---

## 2. Overall Architecture

```
                        ┌─────────────┐
                        │  Checker     │
                        └────┬────────┘
                             │
                    Initiates retrieval request
                             │
                             ▼
                ┌────────────────────┐
                │   Record SP's IP    │
                └────────────────────┘
                             │
                             ▼
     ┌────────────────────────────────────────────┐
     │  Perform IP risk analysis (async or real-time) │
     └────────────────────────────────────────────┘
         │         │               │          │
         ▼         ▼               ▼          ▼
   📡 IPInfo     🛡️ IPQualityScore   🗂️ Local DB    🧠 DNS/WHOIS
  IP ownership     Detect VPN/proxy     Check cloud     Reverse DNS / Registrar
                             │
                             ▼
                 ┌────────────────────┐
                 │  Risk Score Compute  │
                 └────────────────────┘
                             │
                             ▼
         Save result to local cache and MongoDB
```

---

## 3. Technical Details and Implementation

### 1. Obtaining the SP's Public IP

- **Method 1:** Get HTTP multiaddr using `/fil/retrieval/transport/1.0.0` protocol - SP must support this protocol and return HTTP endpoint.
  Boost or compatible implementations are acceptable.

- **Method 2:** SP exposes its IP via libp2p communication.

- **Method 3:** Capture source IP from `/ipfs`, `/piece` HTTP requests.

---

### 2. Combine Multiple API Checks

#### ✅ IPInfo.io (Basic Ownership Info)
- Fields: `org`, `asn`, `country`, `hosting`
- If `hosting=true` or `org` contains AWS/GCP/Azure, mark as cloud node.

#### ✅ IPQualityScore (Anonymous Network Detection)
- Fields: `vpn`, `proxy`, `tor`, `active_vpn`, `host`, `fraud_score`
- If `fraud_score > 75` and vpn/proxy detected → high risk
- `tor=true` → critical risk

#### ✅ Local Database Check
- Use local `IP-to-ASN` or `IP-to-Location` databases
- Determine if IP is from known IDC/VPN ranges
- Can use open datasets such as MaxMind or IP2Location-Lite

#### ✅ DNS Reverse and Whois Lookup
- Reverse DNS contains: `cloud`, `vpn`, `tor`, `ec2`
- Whois registrar matches cloud provider (Amazon, Google, Alibaba, etc.)

---

## 4. Caching Strategy

- All detection results are cached in MongoDB using IP as key:

```json
{
  "ip": "13.214.88.XX",
  "timestamp": "2025-06-11T12:00:00Z",
  "vpn": true,
  "proxy": false,
  "tor": false,
  "fraud_score": 85,
  "asn": "AS16509",
  "provider": "Amazon",
  "source": "IPQS + IPInfo",
  "risk_level": "high"
}
```

- TTL set to 24 hours; prefer cached values for repeated queries
- Background tasks refresh cache data asynchronously

---

## 5. Risk Scoring System

| Risk Factor         | Condition                  | Score |
|---------------------|----------------------------|-------|
| VPN Detection       | `vpn=true`                 | +40   |
| Proxy Detection     | `proxy=true`               | +20   |
| TOR Detection       | `tor=true`                 | +60   |
| Cloud Hosting       | `hosting=true` or cloud ASN| +30   |
| Fraud Score         | `fraud_score > 75`         | +30   |
| Location Anomaly    | Geolocation mismatch       | +15   |
| Data Center Source  | ASN in known IDCs          | +20   |

**Rule:** Total score > 60 → considered untrusted IP (blacklist or downgrade)

---

## 6. Deployment

- 🛠 Deployed as **asynchronous service module**:
  - Log IP after retrieval
  - Batch check via Kafka, cron job, or consumer queue
  - Real-time mode supported for high-value requests
  - Visual dashboard using Prometheus + Grafana

---

## 7. Security

- Do not store sensitive IP-related transaction data
- Risk flags only; full details saved in private MongoDB
- Link SP’s libp2p PeerID ↔ IP for deeper analysis

---

## 8. MongoDB Schema: `sp_ip_risk_info` Collection

### ✅ Example Document

```json
{
  "ip": "13.214.88.23",
  "timestamp": "2025-06-11T14:32:20Z",
  "checker_peer_id": "12D3KooWRzabcXYZ...",
  "provider_id": "f0123456",
  "risk_sources": {
    "ipqualityscore": {
      "vpn": true,
      "proxy": false,
      "tor": false,
      "active_vpn": true,
      "fraud_score": 88
    },
    "ipinfo": {
      "asn": "AS16509",
      "org": "Amazon.com, Inc.",
      "country": "SG",
      "hosting": true
    },
    "local_db": {
      "asn_name": "AMAZON-AES",
      "is_cloud_provider": true,
      "is_idc": true
    },
    "dns_reverse": "ec2-13-214-88-23.ap-southeast-1.compute.amazonaws.com",
    "whois": {
      "netname": "AMAZON-EC2-AP",
      "descr": "Amazon AWS",
      "country": "SG"
    }
  },
  "summary": {
    "risk_level": "high",
    "total_score": 85,
    "category": ["vpn", "cloud"]
  },
  "geo": {
    "country": "SG",
    "region": "Singapore",
    "city": "Singapore"
  },
  "cache_ttl": "2025-06-12T14:32:20Z"
}
```

---

### Field Descriptions

| Field Name        | Type     | Description                               |
|-------------------|----------|-------------------------------------------|
| ip                | string   | Public IP address (primary key)           |
| timestamp         | datetime | First or latest detection time            |
| checker_peer_id   | string   | PeerID of the checking node (optional)    |
| provider_id       | string   | Target SP’s miner ID (e.g., `f01234`)     |
| risk_sources      | object   | Detection source breakdown                |
| summary           | object   | Risk level and score summary              |
| geo               | object   | Geolocation info                          |
| cache_ttl         | datetime | TTL for cached detection result           |

---
