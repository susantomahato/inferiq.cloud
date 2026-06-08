# inferiq.cloud — Infrastructure & Setup Documentation

## Overview

This document covers the complete setup of `inferiq.cloud` including static website hosting, HTTPS via CloudFront, custom email, and PagerDuty alerting for H1B visa slot notifications.

---

## 1. AWS CLI Setup

**Installation:** Official AWS CLI v2 (standalone binary, not Homebrew)

```bash
# Download and install
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o /tmp/AWSCLIV2.pkg
sudo installer -pkg /tmp/AWSCLIV2.pkg -target /

# Verify
aws --version

# Configure credentials
aws configure
```

**Account:** `susanto.mahato` — Account ID: `016829298065`

**References:**
- [AWS CLI v2 Install Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [AWS CLI Configuration](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)

---

## 2. S3 Static Website Hosting

**Bucket:** `inferiq.cloud` (region: `us-east-2`)

- Static website hosting enabled
- Index document: `index.html`
- Error document: `error.html`
- 404 routing rule → redirects to `index.html` (SPA support)
- Public access block: disabled
- Bucket policy: public `s3:GetObject` on all objects

**Website endpoint:** `inferiq.cloud.s3-website.us-east-2.amazonaws.com`

**Deploy new files:**
```bash
aws s3 cp index.html s3://inferiq.cloud/
# or sync entire folder
aws s3 sync ./dist s3://inferiq.cloud/
```

**References:**
- [S3 Static Website Hosting](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html)

---

## 3. ACM SSL Certificate

Two certificates exist:

| Region | ARN | Status | Used By |
|--------|-----|--------|---------|
| `us-east-2` | `arn:aws:acm:us-east-2:016829298065:certificate/a16b5fc8-4077-4da1-b11c-92e95cc69433` | ISSUED | Not in use |
| `us-east-1` | `arn:aws:acm:us-east-1:016829298065:certificate/b71d6d70-ab21-4a90-837b-ae678b5d34ce` | ISSUED | CloudFront |

> **Note:** CloudFront requires the ACM certificate to be in `us-east-1`. Always use the `us-east-1` cert for CloudFront.

Covers: `inferiq.cloud` + `www.inferiq.cloud`
Validation: DNS (CNAMEs in Route 53)

**References:**
- [ACM with CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cnames-and-https-requirements.html)

---

## 4. CloudFront Distribution

**Active Distribution:** `E2DZJWSPK3DAQH`
- Domain: `d17g1mv2wb6hwa.cloudfront.net`
- Aliases: `inferiq.cloud`, `www.inferiq.cloud`
- Origin: S3 website endpoint (`inferiq.cloud.s3-website.us-east-2.amazonaws.com`)
- Viewer protocol: HTTP → HTTPS redirect
- HTTPS: TLSv1.2_2021
- Default root: `index.html`
- 404 → returns `index.html` with 200 (SPA routing)
- Pricing: Pay-as-you-go
- Compression: enabled
- HTTP/2: enabled, IPv6: enabled

**Disabled Distribution (pending deletion July 1):** `EOI1Q5K2UIAH6`
- Status: Disabled, Pricing: Free (WAF plan cancelled)
- Safe to delete after June 30 billing cycle ends

**Invalidate cache after deploy:**
```bash
aws cloudfront create-invalidation --distribution-id E2DZJWSPK3DAQH --paths "/*"
```

**References:**
- [CloudFront with S3](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/GettingStartedCreateDistrib.html)

---

## 5. Route 53 DNS

**Hosted Zone:** `inferiq.cloud` — Zone ID: `Z0241735HP71DFG95JM0`

| Record | Type | Value |
|--------|------|-------|
| `inferiq.cloud` | A (ALIAS) | `d17g1mv2wb6hwa.cloudfront.net` |
| `www.inferiq.cloud` | A (ALIAS) | `d17g1mv2wb6hwa.cloudfront.net` |
| `inferiq.cloud` | MX | `10 mx1.privateemail.com`, `10 mx2.privateemail.com` |
| `inferiq.cloud` | TXT | `v=spf1 include:spf.privateemail.com ~all` |
| `_202ac6d08c073714950e31636b936e8f.inferiq.cloud` | CNAME | ACM validation |
| `_ff7826172e6d2bc934c2c997c072029f.www.inferiq.cloud` | CNAME | ACM validation |

**List all records:**
```bash
aws route53 list-resource-record-sets --hosted-zone-id Z0241735HP71DFG95JM0
```

**References:**
- [Route 53 Alias Records](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html)

---

## 6. Custom Email — Namecheap Private Email

**Plan:** Launch ($14.88/year)
**Login:** [privateemail.com](https://privateemail.com)

| Mailbox | Address |
|---------|---------|
| Primary | `susanto@inferiq.cloud` |
| Alias | `dalima@inferiq.cloud` |
| Alias | `pankaj@inferiq.cloud` |

> Aliases share the same inbox as `susanto@inferiq.cloud`. They can receive email but cannot send or log in independently.

**MX Records (in Route 53):**
- `10 mx1.privateemail.com`
- `10 mx2.privateemail.com`

**References:**
- [Namecheap Private Email Docs](https://www.namecheap.com/support/knowledgebase/subcategory/34/private-email/)
- [Private Email with External DNS](https://www.namecheap.com/support/knowledgebase/article.aspx/9661/2208/how-to-set-up-mx-records-for-private-email/)

---

## 7. PagerDuty Setup

**Purpose:** Instant alerts when H1B visa slot emails arrive in Gmail

**Account:** 14-day free trial
**Login:** [app.pagerduty.com](https://app.pagerduty.com)

**Users:**

| Name | Email | Role |
|------|-------|------|
| susanto Mahato | `susanto@inferiq.cloud` | Account Owner |
| Dalima | `dalima@inferiq.cloud` | Responder |
| Pankaj | `pankaj@inferiq.cloud` | Responder |

**Service:** `H1B Visa Alerts`
**Integration:** Events API V2

> Get the Integration Key from: Services → H1B Visa Alerts → Integrations → Events API V2

**References:**
- [PagerDuty Events API V2](https://developer.pagerduty.com/docs/ZG9jOjExMDI5NTgw-events-api-v2-overview)
- [PagerDuty Notification Rules](https://support.pagerduty.com/docs/user-profile#notification-rules)

---

## 8. Gmail → PagerDuty Alert (Google Apps Script)

**Purpose:** Monitors Gmail inbox every 1 minute for H1B slot alert emails → fires PagerDuty alert

**Setup:**
1. Go to [script.google.com](https://script.google.com)
2. Create new project named `H1B Visa Alerts`
3. Paste the script (see below)
4. Set trigger: Time-driven → Minutes timer → Every minute
5. Authorize Gmail and URL Fetch permissions

**Script template:**
```javascript
const PAGERDUTY_INTEGRATION_KEY = 'YOUR_INTEGRATION_KEY_HERE';
const SEARCH_QUERY = 'from:SENDER_EMAIL subject:SUBJECT_KEYWORD is:unread';

function checkGmailAndAlert() {
  const threads = GmailApp.search(SEARCH_QUERY, 0, 5);
  if (threads.length === 0) return;

  threads.forEach(thread => {
    const message = thread.getMessages()[0];
    const subject = message.getSubject();
    const body = message.getPlainBody().substring(0, 500);

    triggerPagerDuty(subject, body);
    thread.markRead();
  });
}

function triggerPagerDuty(subject, body) {
  const payload = {
    routing_key: PAGERDUTY_INTEGRATION_KEY,
    event_action: 'trigger',
    payload: {
      summary: 'H1B Slot Alert: ' + subject,
      severity: 'critical',
      source: 'Gmail',
      custom_details: { email_body: body }
    }
  };

  UrlFetchApp.fetch('https://events.pagerduty.com/v2/enqueue', {
    method: 'post',
    contentType: 'application/json',
    payload: JSON.stringify(payload)
  });
}
```

**References:**
- [Google Apps Script](https://script.google.com)
- [Apps Script GmailApp Docs](https://developers.google.com/apps-script/reference/gmail/gmail-app)
- [PagerDuty Events API V2](https://developer.pagerduty.com/api-reference/YXBpOjI3NDgyNjU-pager-duty-v2-inbound-integration)

---

## 9. Monthly Cost Estimate

| Service | Cost/month |
|---------|-----------|
| S3 storage + requests | ~$0.00 |
| CloudFront (pay-as-you-go, low traffic) | ~$0.00–$0.10 |
| Route 53 hosted zone | $0.50 |
| ACM certificates | Free |
| EC2 `c7i-flex.large` (stopped, EBS 20GB) | ~$1.60 |
| Namecheap Private Email | ~$1.24 ($14.88/yr) |
| PagerDuty (after trial) | TBD |
| **Total** | **~$3.50–$4.00/month** |

> The stopped EC2 instance (`i-0f70b91b0927ee3bc`) still charges ~$1.60/month for its EBS volume. Consider terminating if unused.

---

## 10. Pending Actions

- [ ] Delete CloudFront distribution `EOI1Q5K2UIAH6` after **July 1, 2026**:
  ```bash
  ETAG=$(aws cloudfront get-distribution --id EOI1Q5K2UIAH6 --query 'ETag' --output text)
  aws cloudfront delete-distribution --id EOI1Q5K2UIAH6 --if-match $ETAG
  ```
- [ ] Dalima and Pankaj accept PagerDuty invite emails
- [ ] Configure PagerDuty notification rules (SMS/phone) for all 3 users
- [ ] Complete Google Apps Script setup with actual Integration Key + H1B email filter
- [ ] Decide whether to terminate stopped EC2 instance to save $1.60/month
- [ ] PagerDuty free trial expires — choose a paid plan or downgrade before trial ends
