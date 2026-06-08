# inferiq.cloud

**inferiq.cloud** is a personal project exploring AI-powered intelligence.

🌐 Live site: [https://inferiq.cloud](https://inferiq.cloud)

This repository documents the cloud infrastructure, custom email setup, and automated email alerting pipeline built to support the project.

---

## Infrastructure Overview

| Service | Purpose |
|---------|---------|
| AWS S3 | Static website hosting |
| AWS CloudFront | CDN + HTTPS |
| AWS Route 53 | DNS management |
| AWS ACM | SSL/TLS certificate |
| Namecheap Private Email | Custom domain email (`@inferiq.cloud`) |
| PagerDuty | Incident alerting |
| Google Apps Script | Gmail inbox monitoring → PagerDuty trigger |

---

## 1. Static Website (S3 + CloudFront + HTTPS)

### Prerequisites
- AWS CLI v2 installed — [Install Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- AWS credentials configured — run `aws configure`

### Deploy website
```bash
# Upload a single file
aws s3 cp index.html s3://inferiq.cloud/

# Or sync an entire build folder
aws s3 sync ./dist s3://inferiq.cloud/
```

### Invalidate CloudFront cache after deploy
```bash
aws cloudfront create-invalidation --distribution-id E2DZJWSPK3DAQH --paths "/*"
```

### Useful links
- [S3 Static Website Hosting](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html)
- [CloudFront with S3](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/GettingStartedCreateDistrib.html)
- [ACM Certificate with CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cnames-and-https-requirements.html)

---

## 2. Custom Email (`@inferiq.cloud`)

Email is hosted via **Namecheap Private Email** with MX records pointing from Route 53.

### MX Records (Route 53)
```
10 mx1.privateemail.com
10 mx2.privateemail.com
```

### SPF Record (Route 53 TXT)
```
v=spf1 include:spf.privateemail.com ~all
```

### Manage DNS records
```bash
aws route53 list-resource-record-sets --hosted-zone-id Z0241735HP71DFG95JM0
```

### Useful links
- [Namecheap Private Email](https://privateemail.com)
- [Namecheap Private Email Docs](https://www.namecheap.com/support/knowledgebase/subcategory/34/private-email/)

---

## 3. Email → PagerDuty Alerting (Google Apps Script)

Any email arriving in the monitored Gmail inbox triggers an immediate PagerDuty alert. The script runs every minute via a time-driven trigger.

### Setup steps
1. Go to [script.google.com](https://script.google.com)
2. Create a new project
3. Paste the script below — fill in `PAGERDUTY_INTEGRATION_KEY` and `SEARCH_QUERY`
4. Add trigger: **Time-driven → Minutes timer → Every minute**
5. Authorize Gmail and URL Fetch permissions on first run

### Script
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
      summary: subject,
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

### Get your PagerDuty Integration Key
1. Log into [app.pagerduty.com](https://app.pagerduty.com)
2. Go to **Services → Your Service → Integrations**
3. Add **Events API V2** integration
4. Copy the **Integration Key**

### Useful links
- [Google Apps Script Docs](https://developers.google.com/apps-script)
- [GmailApp Reference](https://developers.google.com/apps-script/reference/gmail/gmail-app)
- [PagerDuty Events API V2](https://developer.pagerduty.com/docs/ZG9jOjExMDI5NTgw-events-api-v2-overview)

---

## 4. Pending Actions

- [ ] Delete disabled CloudFront distribution after July 1, 2026:
  ```bash
  ETAG=$(aws cloudfront get-distribution --id EOI1Q5K2UIAH6 --query 'ETag' --output text)
  aws cloudfront delete-distribution --id EOI1Q5K2UIAH6 --if-match $ETAG
  ```
- [ ] Complete Google Apps Script setup with actual Integration Key and email filter
- [ ] Configure PagerDuty notification rules (SMS/phone) for all users
