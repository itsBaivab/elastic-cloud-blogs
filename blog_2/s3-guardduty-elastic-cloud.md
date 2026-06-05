# Detect S3 Threats in Real Time with Amazon GuardDuty and Elastic Cloud

Your S3 buckets hold your most sensitive data — database backups, user uploads, application configs. AWS GuardDuty watches those buckets continuously and fires a finding the moment something looks wrong: an unusual IP pulling objects at 3am, a root account touching production data, an API call from a Tor exit node.

The problem is GuardDuty findings stay inside the AWS console by default. This walkthrough ships them into Elastic Cloud so you can search, correlate, and alert across every finding from every account in one place — with no agent to manage.

## What you will build

```
Amazon GuardDuty (S3 Protection enabled)
  └── Findings stored in GuardDuty
        └── Elastic GuardDuty Integration (agentless, polls API every 1m)
              └── Kibana: search findings, dashboards, alerts
```

One Elastic agentless integration calling the GuardDuty API directly — zero servers, zero agents, zero infrastructure to manage.

---

## Table of contents

- [Prerequisites](#prerequisites)
- [Step 1: Provision Elastic Cloud from AWS Marketplace](#step-1-provision-elastic-cloud-from-aws-marketplace)
- [Step 2: Generate an Elastic API Key](#step-2-generate-an-elastic-api-key)
- [Step 3: Create the Findings S3 Bucket](#step-3-create-the-findings-s3-bucket)
- [Step 4: Enable GuardDuty and S3 Protection](#step-4-enable-guardduty-and-s3-protection)
- [Step 5: Export GuardDuty Findings to the S3 Bucket](#step-5-export-guardduty-findings-to-the-s3-bucket)
- [Step 6: Create an SQS Queue for Notifications](#step-6-create-an-sqs-queue-for-notifications)
- [Step 7: Create an IAM User for Elastic](#step-7-create-an-iam-user-for-elastic)
- [Step 7: Configure the GuardDuty Integration in Kibana](#step-7-configure-the-guardduty-integration-in-kibana)
- [Step 8: Generate Sample Findings to Test the Pipeline](#step-8-generate-sample-findings-to-test-the-pipeline)
- [Step 9: Explore Findings in Kibana](#step-9-explore-findings-in-kibana)
- [What findings to watch for](#what-findings-to-watch-for)
- [FAQs](#faqs)

---

## Prerequisites

- An AWS account with GuardDuty not yet enabled (free 30-day trial)
- AWS CLI configured (`aws configure`)
- A browser

---

## Step 1: Provision Elastic Cloud from AWS Marketplace

1. Open the [AWS Console](https://console.aws.amazon.com) and navigate to **AWS Marketplace**.
2. Search for **Elastic Cloud (Elasticsearch Service)** and open the listing.

![Elastic Cloud listing on AWS Marketplace](images/aws_marketplace_elastic_listing.png)

3. Click **View purchase options**. Elastic Cloud is usage-based — no upfront commit and a free trial is included.

![AWS Marketplace subscribe page](images/aws_marketplace_subscribe.png)

4. Click **Subscribe**, then **Set up your account**. You will land on the Elastic Cloud login page with a banner confirming the AWS billing link is active.

![Elastic Cloud login via AWS Marketplace](images/elastic_cloud_login_aws.png)

5. Your AWS Marketplace agreement shows **Active**. Click **Set up your account** to continue.

![AWS Marketplace agreement — Active](images/aws_marketplace_agreement.png)

6. In the Elastic Cloud portal, click **Create serverless project**.

![Create your first serverless project](images/elastic_create_serverless_project.png)

7. Select **Elastic for Observability**.

![Select Elastic for Observability](images/elastic_choose_observability.png)

8. Name the project `guardduty-s3`, set the cloud provider to **Amazon Web Services**, choose the same region as your GuardDuty detector (e.g. `us-east-1`), and click **Create**.

When provisioning completes, click **Open project**.

![Project is ready](images/elastic_project_ready.png)

9. On the project overview, copy your **Elasticsearch endpoint** for reference.

![Elastic project overview](images/elastic_project_overview.png)

---

## Step 2: Generate an Elastic API Key

In Kibana, navigate to **☰ → Stack Management → Security → API Keys → Create API key**.

Name it `guardduty-integration` and click **Create API key**.

![API key created in Kibana](images/elastic_api_key_created.png)

Copy the encoded key immediately — it is shown only once.

---

## Step 3: Create the Findings S3 Bucket

Before enabling GuardDuty, create the S3 bucket that will receive all exported findings. This bucket is the central store that Elastic reads from.

```bash
# Set your account ID and region
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=us-east-1
FINDINGS_BUCKET="guardduty-findings-${ACCOUNT_ID}-${REGION}"

echo "Creating bucket: $FINDINGS_BUCKET"

# Create the bucket
aws s3api create-bucket \
  --bucket $FINDINGS_BUCKET \
  --region $REGION

# Block all public access
aws s3api put-public-access-block \
  --bucket $FINDINGS_BUCKET \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

echo "Bucket ready: $FINDINGS_BUCKET"
```

Now allow GuardDuty to write findings into it:

```bash
aws s3api put-bucket-policy \
  --bucket $FINDINGS_BUCKET \
  --policy "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [
      {
        \"Sid\": \"AllowGuardDutyPutFindings\",
        \"Effect\": \"Allow\",
        \"Principal\": { \"Service\": \"guardduty.amazonaws.com\" },
        \"Action\": \"s3:PutObject\",
        \"Resource\": \"arn:aws:s3:::${FINDINGS_BUCKET}/*\",
        \"Condition\": {
          \"StringEquals\": { \"aws:SourceAccount\": \"${ACCOUNT_ID}\" }
        }
      },
      {
        \"Sid\": \"AllowGuardDutyGetBucketLocation\",
        \"Effect\": \"Allow\",
        \"Principal\": { \"Service\": \"guardduty.amazonaws.com\" },
        \"Action\": \"s3:GetBucketLocation\",
        \"Resource\": \"arn:aws:s3:::${FINDINGS_BUCKET}\"
      }
    ]
  }"

echo "Bucket policy applied — GuardDuty can now write to this bucket"
```

> **Placeholder:** Add a screenshot of the S3 bucket in the AWS console showing **Block Public Access: On** and the bucket policy applied.

---

## Step 4: Enable GuardDuty and S3 Protection

### 4a. Enable GuardDuty

```bash
# Enable GuardDuty in your region
aws guardduty create-detector \
  --enable \
  --region us-east-1 \
  --finding-publishing-frequency FIFTEEN_MINUTES

# Save the detector ID — needed in the next steps
DETECTOR_ID=$(aws guardduty list-detectors \
  --region us-east-1 \
  --query 'DetectorIds[0]' \
  --output text)
echo "Detector ID: $DETECTOR_ID"
```

### 4b. Enable S3 Protection

S3 Protection instructs GuardDuty to monitor all API calls to your S3 buckets — detecting policy misconfigurations, unusual access patterns, and data exfiltration attempts.

```bash
aws guardduty update-detector \
  --detector-id $DETECTOR_ID \
  --region us-east-1 \
  --data-sources '{"S3Logs":{"Enable":true}}'

# Verify S3 Protection is on
aws guardduty get-detector \
  --detector-id $DETECTOR_ID \
  --region us-east-1 \
  --query 'DataSources.S3Logs'
# Expect: {"Status": "ENABLED"}
```

> **Placeholder:** Add a screenshot of the GuardDuty console showing S3 Protection status **Enabled**.

---

## Step 5: Export GuardDuty Findings to the S3 Bucket

Now point GuardDuty at the bucket you created in Step 3 so it exports every finding automatically.

```bash
# Use the same variables from Step 3
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=us-east-1
FINDINGS_BUCKET="guardduty-findings-${ACCOUNT_ID}-${REGION}"
DETECTOR_ID=$(aws guardduty list-detectors \
  --region $REGION \
  --query 'DetectorIds[0]' \
  --output text)

aws guardduty update-findings-export-settings \
  --detector-id $DETECTOR_ID \
  --region $REGION \
  --destination-type S3 \
  --destination-properties "{
    \"DestinationArn\": \"arn:aws:s3:::${FINDINGS_BUCKET}\",
    \"KmsKeyArn\": \"\"
  }" \
  --finding-publishing-frequency FIFTEEN_MINUTES

echo "GuardDuty will now export findings to: $FINDINGS_BUCKET"
```

**What `FIFTEEN_MINUTES` means:** GuardDuty batches new findings and pushes them to S3 every 15 minutes. You can change this to `ONE_HOUR` or `SIX_HOURS` to reduce S3 write costs if low latency is not a requirement.

> **Placeholder:** Add a screenshot of the GuardDuty console → Settings → Findings export options showing the S3 bucket configured.

---

## Step 6: Create an SQS Queue for Notifications

Elastic polls SQS rather than listing the S3 bucket directly — this is more efficient and avoids missing findings during high-volume periods.

### 6a. Create the queue

```bash
QUEUE_URL=$(aws sqs create-queue \
  --queue-name guardduty-findings-queue \
  --region $REGION \
  --query 'QueueUrl' \
  --output text)

QUEUE_ARN=$(aws sqs get-queue-attributes \
  --queue-url $QUEUE_URL \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' \
  --output text)

echo "Queue URL: $QUEUE_URL"
echo "Queue ARN: $QUEUE_ARN"
```

### 5b. Allow S3 to send notifications to the queue

```bash
aws sqs set-queue-attributes \
  --queue-url $QUEUE_URL \
  --attributes "{
    \"Policy\": \"{\\\"Version\\\":\\\"2012-10-17\\\",\\\"Statement\\\":[{\\\"Sid\\\":\\\"AllowS3SendMessage\\\",\\\"Effect\\\":\\\"Allow\\\",\\\"Principal\\\":{\\\"Service\\\":\\\"s3.amazonaws.com\\\"},\\\"Action\\\":\\\"sqs:SendMessage\\\",\\\"Resource\\\":\\\"${QUEUE_ARN}\\\",\\\"Condition\\\":{\\\"ArnLike\\\":{\\\"aws:SourceArn\\\":\\\"arn:aws:s3:::${FINDINGS_BUCKET}\\\"}}}]}\"
  }"
```

### 5c. Enable S3 event notifications to SQS

```bash
aws s3api put-bucket-notification-configuration \
  --bucket $FINDINGS_BUCKET \
  --notification-configuration "{
    \"QueueConfigurations\": [
      {
        \"QueueArn\": \"${QUEUE_ARN}\",
        \"Events\": [\"s3:ObjectCreated:*\"]
      }
    ]
  }"
```

> **Placeholder:** Add a screenshot of the S3 bucket → Properties → Event notifications showing the SQS destination.

---

## Step 6: Create an IAM User for Elastic

The integration needs read access to S3 and SQS only. Create a least-privilege user.

### 6a. Create the user

```bash
aws iam create-user --user-name elastic-guardduty-reader
```

### 6b. Attach the policy

Because the integration polls the GuardDuty API directly, the IAM user needs GuardDuty read permissions in addition to S3/SQS access:

```bash
DETECTOR_ID=$(aws guardduty list-detectors --region $REGION --query 'DetectorIds[0]' --output text)

aws iam put-user-policy \
  --user-name elastic-guardduty-reader \
  --policy-name ElasticGuardDutyReadOnly \
  --policy-document "{
    \"Version\": \"2012-10-17\",
    \"Statement\": [
      {
        \"Sid\": \"GuardDutyAPIRead\",
        \"Effect\": \"Allow\",
        \"Action\": [
          \"guardduty:GetFindings\",
          \"guardduty:ListFindings\",
          \"guardduty:GetDetector\",
          \"guardduty:ListDetectors\"
        ],
        \"Resource\": \"arn:aws:guardduty:${REGION}:${ACCOUNT_ID}:detector/${DETECTOR_ID}\"
      },
      {
        \"Sid\": \"ReadFindingsBucket\",
        \"Effect\": \"Allow\",
        \"Action\": [
          \"s3:GetObject\",
          \"s3:ListBucket\"
        ],
        \"Resource\": [
          \"arn:aws:s3:::${FINDINGS_BUCKET}\",
          \"arn:aws:s3:::${FINDINGS_BUCKET}/*\"
        ]
      },
      {
        \"Sid\": \"PollFindingsQueue\",
        \"Effect\": \"Allow\",
        \"Action\": [
          \"sqs:ReceiveMessage\",
          \"sqs:DeleteMessage\",
          \"sqs:GetQueueAttributes\",
          \"sqs:ChangeMessageVisibility\"
        ],
        \"Resource\": \"${QUEUE_ARN}\"
      }
    ]
  }"
```

### 6c. Generate access keys

```bash
aws iam create-access-key \
  --user-name elastic-guardduty-reader \
  --query 'AccessKey.{KeyId:AccessKeyId,Secret:SecretAccessKey}' \
  --output table
```

Save the **KeyId** and **Secret** — you will enter them in Step 7.

---

## Step 7: Configure the GuardDuty Integration in Kibana

### 7a. Open the integration

1. In Kibana, go to **☰ → Add Observability data → Cloud → AWS → View collection**.

![Kibana Add Observability Data — Cloud provider selection](images/kibana_add_data_cloud.png)

2. Search for **aws guard** in the integration search box. Click **Amazon GuardDuty**.

![Kibana search showing Amazon GuardDuty integration](images/kibana_search_guardduty.png)

3. On the integration overview page, click **Add Amazon GuardDuty**.

![Amazon GuardDuty integration page in Kibana](images/kibana_guardduty_integration.png)

### 7b. Fill in the integration form

The form has several sections. Here is exactly what to enter in each field:

**Integration settings**

| Field | Value |
|---|---|
| Integration name | `guardduty-s3` |
| Description | *(optional)* |

**AWS credentials**

| Field | Value | Where to get it |
|---|---|---|
| Access Key ID | `AKIAQSCIQUH6UTOF7RCT` | Created in Step 6c via `aws iam create-access-key` |
| Secret Access Key | *(from Step 6c output)* | Shown once when key was created — save it immediately |
| Session Token | *(leave blank)* | Only needed for temporary credentials via STS/AssumeRole |

![AWS credentials for the integration](images/elastic_aws_credentials.png)

> **Tip:** If you lost the secret key, you cannot retrieve it. Delete the old key with `aws iam delete-access-key --user-name elastic-guardduty-reader --access-key-id <KeyId>` and run `aws iam create-access-key` again to generate a new pair.

**Deployment options**

Select **Agentless (Beta)** — this means Elastic manages the collector entirely; no EC2, no agent to install.

![Agentless deployment option](images/elastic_agentless_option.png)

**Collect Amazon GuardDuty logs via API**

Toggle this **ON**. This is the direct API collection mode — Elastic calls the GuardDuty `GetFindings` API on your behalf using the credentials above.

| Field | Value | Where to get it |
|---|---|---|
| Interval | `1m` | How often Elastic polls for new findings. Keep at `1m` for near-real-time. |
| Initial Interval | `24h` | How far back to pull findings on first run. `24h` backfills the last day. |
| **Detector ID** | `18cf4c78b3244ce491e6e890fab91845` | See below ↓ |
| **AWS Region** | `us-east-1` | The region where you enabled GuardDuty in Step 4 |
| Preserve original event | OFF *(default)* | Turn ON only if you want the raw GuardDuty JSON preserved alongside the ECS-mapped fields |

**How to find your Detector ID:**

```bash
aws guardduty list-detectors --region us-east-1 --query 'DetectorIds[0]' --output text
# Output: 18cf4c78b3244ce491e6e890fab91845
```

Or in the AWS Console: **GuardDuty → Settings** — the Detector ID is shown at the top of the page.

Here is the fully populated form before clicking Save:

![Kibana GuardDuty integration form — fully populated](images/kibana_guardduty_form_filled.png)

### 7c. Save and deploy

Click **Save and continue**. Within a minute you will see:

![Agentless deployment successful](images/elastic_agentless_success.png)

---

## Step 8: Generate Sample Findings to Test the Pipeline

GuardDuty won't produce real findings immediately — it needs time to establish baselines. Use `create-sample-findings` to inject synthetic findings right now and verify data flows into Kibana:

```bash
DETECTOR_ID=$(aws guardduty list-detectors --region us-east-1 --query 'DetectorIds[0]' --output text)

aws guardduty create-sample-findings \
  --detector-id $DETECTOR_ID \
  --region us-east-1

echo "Sample findings injected — wait ~1 minute then check Kibana"
```

This creates ~40 sample findings covering all GuardDuty finding types. They are clearly marked as samples and will not trigger real alerts. Within 1 minute the agentless integration polls the API and findings appear in Kibana.

---

## Step 9: Explore Findings in Kibana

### 9a. Create a data view

Go to **☰ → Stack Management → Data Views → Create data view**.

- **Name**: `GuardDuty Findings`
- **Index pattern**: `logs-aws.guardduty-*`
- **Timestamp field**: `@timestamp`

![Create data view for GuardDuty findings](images/kibana_create_data_view.png)

### 9b. Open Discover

Go to **☰ → Discover**, select **GuardDuty Findings**, and set the time range to **Last 24 hours**.

Each document is one GuardDuty finding — with severity, finding type, affected resource (bucket name, object key), source IP, and AWS account.

![Kibana Discover showing GuardDuty findings](images/kibana_discover_logs.png)

### Useful KQL queries

**High and critical severity findings:**
```kql
aws.guardduty.severity >= 7
```

**All S3-related findings:**
```kql
aws.guardduty.type: *S3*
```

**Findings involving a specific bucket:**
```kql
aws.guardduty.resource.s3BucketDetails.name: "my-sensitive-bucket"
```

**Findings from outside your expected regions:**
```kql
NOT cloud.region: ("us-east-1" OR "us-west-2")
```

**Tor exit node or anonymous IP findings:**
```kql
aws.guardduty.type: *TorIPCaller* OR aws.guardduty.type: *AnonymousIPCaller*
```

---

## What findings to watch for

| Finding type | What it means | Severity |
|---|---|---|
| `Policy:S3/BucketBlockPublicAccessDisabled` | Someone turned off block public access | Medium |
| `Stealth:S3/ServerAccessLoggingDisabled` | Logging disabled — attacker covering tracks | Medium |
| `UnauthorizedAccess:S3/TorIPCaller` | S3 access from Tor exit node | High |
| `Discovery:S3/BucketEnumeration` | Someone listing your buckets | Low–Medium |
| `Exfiltration:S3/ObjectRead.Unusual` | Unusual object access pattern | High |
| `Impact:S3/ObjectDelete.Unusual` | Mass deletions — potential ransomware | High |
| `CredentialAccess:S3/AnomalousBehavior` | Credential anomaly accessing S3 | High |

Set a **Kibana threshold alert** on `aws.guardduty.severity >= 7` with a 5-minute window to get paged immediately on high-severity findings.

---

## FAQs

**Is this truly agentless?**

Yes. The Elastic GuardDuty integration supports agentless deployment on Elastic Cloud Serverless — Elastic manages the collector for you. No EC2, no Lambda, nothing to patch.

**How soon do findings appear after an event?**

GuardDuty generates findings within minutes of detecting a threat. The export to S3 runs every 15 minutes (configurable to 1 or 6 hours for lower cost). Elastic polls SQS continuously, so end-to-end latency is roughly 15 minutes at the tightest setting.

**How much does GuardDuty cost?**

GuardDuty S3 Protection pricing is based on the number of S3 API calls monitored (~$0.40 per 1M events). A 30-day free trial is available for new accounts. The Elastic Cloud cost depends on ingest volume — a small Observability project starts around $95/month.

**Can I monitor multiple AWS accounts?**

Yes. Enable GuardDuty in each account and configure each to export to the same central S3 findings bucket (using a cross-account bucket policy). One Elastic integration reads all findings in one place.

**What if I already use GuardDuty but haven't enabled S3 Protection?**

Run the `update-detector` command from Step 3b against your existing detector ID. S3 Protection activates immediately with no service disruption.

---

## References

- [Elastic Amazon GuardDuty Integration](https://www.elastic.co/docs/reference/integrations/aws/guardduty)
- [Elastic Agentless Integrations list](https://www.elastic.co/docs/reference/integrations/agentless_integrations)
- [AWS GuardDuty S3 Protection docs](https://docs.aws.amazon.com/guardduty/latest/ug/s3-protection.html)
- [GuardDuty finding types reference](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_finding-types-active.html)
- [Elastic Cloud on AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-voru33wi6xs7k)
