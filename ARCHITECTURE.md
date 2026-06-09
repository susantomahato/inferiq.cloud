# inferiq.cloud — System Architecture

> End-to-end infrastructure, alerting, and analytics pipeline.

---

## ① Website Request Flow

```mermaid
flowchart LR
    A(["🌐 Client Browser\nhttps://inferiq.cloud"])
    B(["🔵 Route 53\nDNS Resolution\nA ALIAS Record"])
    C(["🟣 CloudFront\nCDN + TLS\nE2DZJWSPK3DAQH"])
    D(["🟢 S3 Bucket\nStatic Website\nus-east-2"])
    E(["🔴 ACM Certificate\nSSL/TLS\nus-east-1"])

    A -->|"DNS lookup"| B
    B -->|"Resolves to CF edge"| C
    C -->|"Cache MISS → fetch origin"| D
    E -.-|"Attached to"| C

    style A fill:#1e293b,color:#e2e8f0,stroke:#334155
    style B fill:#6d28d9,color:#fff,stroke:#7c3aed
    style C fill:#5b21b6,color:#fff,stroke:#7c3aed
    style D fill:#3f6212,color:#fff,stroke:#569A31
    style E fill:#9f1239,color:#fff,stroke:#DD344C
```

---

## ② Email Flow

```mermaid
flowchart TD
    A(["✉️ External Sender\n*@inferiq.cloud"])
    B(["🔵 Route 53\nMX + SPF Records"])
    C(["🔴 Namecheap\nPrivate Email\nmx1/mx2.privateemail.com"])
    D(["📬 Mailboxes\nsusanto@inferiq.cloud\ndalima@inferiq.cloud\npankaj@inferiq.cloud"])

    A -->|"MX lookup"| B
    B -->|"Routes to mail server"| C
    C -->|"Delivered"| D

    style A fill:#1e293b,color:#e2e8f0,stroke:#334155
    style B fill:#6d28d9,color:#fff,stroke:#7c3aed
    style C fill:#991b1b,color:#fff,stroke:#DE3723
    style D fill:#991b1b,color:#fff,stroke:#DE3723
```

---

## ③ Email Alert Pipeline

```mermaid
flowchart TD
    A(["📩 Gmail Inbox\nAlert email arrives"])
    B(["⚙️ Google Apps Script\ncheckGmailAndAlert()\nPolls every 1 minute"])
    C(["🚨 PagerDuty\nH1B Visa Alerts Service\nEvents API v2"])
    D(["👤 Susanto Mahato\nAccount Owner"])
    E(["👤 Dalima Mahato\nResponder"])
    F(["👤 Pankaj Mahato\nResponder"])

    A -->|"Unread email found"| B
    B -->|"POST /v2/enqueue"| C
    C -->|"Push / SMS / Call"| D
    C -->|"Push / SMS / Call"| E
    C -->|"Push / SMS / Call"| F

    style A fill:#1e3a5f,color:#fff,stroke:#4285F4
    style B fill:#1e3a5f,color:#fff,stroke:#4285F4
    style C fill:#064e3b,color:#fff,stroke:#06AC38
    style D fill:#064e3b,color:#fff,stroke:#06AC38
    style E fill:#064e3b,color:#fff,stroke:#06AC38
    style F fill:#064e3b,color:#fff,stroke:#06AC38
```

---

## ④ Analytics Pipeline

```mermaid
flowchart LR
    A(["🌐 Client Request\nAny page visit"])
    B(["🟣 CloudFront\nLogs: IP, Country\nStatus, Bytes, UA"])
    C(["🟢 S3 Logs Bucket\ninferiq-cloud-logs\ncf-logs/"])
    D(["🔍 Amazon Athena\ninferiq_analytics\ncloudfront_logs"])
    E(["📊 Insights\nHits · IPs · Countries\nStatus · Pages"])

    A -->|"HTTP request"| B
    B -->|"~15 min batch"| C
    C -->|"Query on-demand"| D
    D -->|"Results"| E

    style A fill:#1e293b,color:#e2e8f0,stroke:#334155
    style B fill:#5b21b6,color:#fff,stroke:#7c3aed
    style C fill:#3f6212,color:#fff,stroke:#569A31
    style D fill:#5b21b6,color:#fff,stroke:#7c3aed
    style E fill:#1e3a5f,color:#fff,stroke:#FF9900
```

---

## ⑤ Deployment Flow

```mermaid
flowchart LR
    A(["💻 Developer\nLocal changes"])
    B(["🐙 GitHub\nsusantomahato/inferiq.cloud"])
    C(["🟢 aws s3 sync\nUpload to S3"])
    D(["🟣 Cache Invalidation\nCloudFront /*"])
    E(["✅ Live\nhttps://inferiq.cloud"])

    A -->|"git push"| B
    B -->|"aws s3 cp/sync"| C
    C -->|"create-invalidation"| D
    D -->|"~2 min propagation"| E

    style A fill:#1e293b,color:#e2e8f0,stroke:#334155
    style B fill:#3b0764,color:#fff,stroke:#6e40c9
    style C fill:#3f6212,color:#fff,stroke:#569A31
    style D fill:#5b21b6,color:#fff,stroke:#7c3aed
    style E fill:#064e3b,color:#fff,stroke:#06AC38
```

---

## ⑥ DNS Record Map

```
inferiq.cloud  (Route 53 — Z0241735HP71DFG95JM0)
│
├── A     inferiq.cloud        →  d17g1mv2wb6hwa.cloudfront.net  (ALIAS)
├── A     www.inferiq.cloud    →  d17g1mv2wb6hwa.cloudfront.net  (ALIAS)
├── MX    inferiq.cloud        →  10 mx1.privateemail.com
│                                 10 mx2.privateemail.com
├── TXT   inferiq.cloud        →  v=spf1 include:spf.privateemail.com ~all
├── CNAME _202ac6d08c...       →  ACM validation (inferiq.cloud)
└── CNAME _ff7826172e...       →  ACM validation (www.inferiq.cloud)
```

---

## ⑦ Monthly Cost Estimate

| Service | Provider | Purpose | Cost/month |
|---------|----------|---------|-----------|
| Route 53 | AWS | DNS + MX + ACM validation | $0.50 |
| CloudFront | AWS | CDN + HTTPS | ~$0.00 |
| S3 (website) | AWS | Static file hosting | ~$0.00 |
| S3 (logs) | AWS | CloudFront access logs | ~$0.01 |
| Athena | AWS | Log analytics queries | ~$0.00 |
| ACM Certificate | AWS | SSL/TLS | Free |
| Private Email | Namecheap | @inferiq.cloud mailboxes | $1.24 |
| Google Apps Script | Google | Gmail → PagerDuty monitor | Free |
| PagerDuty | PagerDuty | Incident alerting | TBD |
| **Total** | | | **~$2.00/month** |

---

## ⑧ Repository Structure

```
inferiq.cloud/
│
├── index.html              # Main website
├── img-robotics.png        # AI robot showcase image
├── img-analytics.svg       # Analytics dashboard illustration
├── img-neural.svg          # Neural network illustration
├── pankaj.png              # Co-founder photo
├── dalima.png              # Co-founder photo
├── README.md               # Setup instructions
├── ARCHITECTURE.md         # This document
└── architecture.html       # Visual architecture (browser)
```
