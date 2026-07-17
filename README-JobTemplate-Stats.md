# Retrieving Quota Stats from Job Template

This document explains how to retrieve quota statistics from the "Get Mailbox Size and Compare Quota" playbook when running it as a job template in AWX/Ansible Tower.

## Stats Captured

The playbook captures the following statistics using `set_stats`:

| Stat Key | Description | Example Value |
|----------|-------------|---------------|
| `mailbox_email` | The mailbox email address | `to@test.com` |
| `message_count` | Number of messages in mailbox | `5` |
| `mailbox_size_bytes` | Total mailbox size in bytes | `75000` |
| `mailbox_size_mb` | Total mailbox size in MB | `0.07` |
| `quota_allocated_bytes` | Allocated quota in bytes | `1048576` |
| `quota_allocated_mb` | Allocated quota in MB | `1` |
| `quota_used_bytes` | Quota used in bytes | `75000` |
| `quota_used_mb` | Quota used in MB | `0.07` |
| `quota_remaining_bytes` | Remaining quota in bytes | `973576` |
| `quota_remaining_mb` | Remaining quota in MB | `0.93` |
| `quota_percentage_used` | Percentage of quota used | `7.15` |
| `is_over_quota` | Whether mailbox is over quota | `false` |
| `status` | Overall status | `WITHIN_LIMITS` or `OVER_QUOTA` |

## Retrieving Stats via AWX/Tower API

### Step 1: Launch the Job Template

```bash
curl -X POST \
  https://your-awx-server/api/v2/job_templates/{template_id}/launch/ \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "extra_vars": {
      "mailbox_email": "to@test.com"
    }
  }'
```

Response will include a `job` ID:
```json
{
  "job": 123,
  "url": "/api/v2/jobs/123/"
}
```

### Step 2: Wait for Job Completion (Poll Status)

```bash
curl -X GET \
  https://your-awx-server/api/v2/jobs/123/ \
  -H "Authorization: Bearer YOUR_TOKEN"
```

Check the `status` field:
```json
{
  "id": 123,
  "status": "successful",
  "finished": "2026-01-28T12:00:00.000Z",
  ...
}
```

### Step 3: Retrieve Artifacts (Stats)

Once the job is complete, retrieve the artifacts:

```bash
curl -X GET \
  https://your-awx-server/api/v2/jobs/123/ \
  -H "Authorization: Bearer YOUR_TOKEN"
```

The stats will be in the `artifacts` field:
```json
{
  "id": 123,
  "status": "successful",
  "artifacts": {
    "mailbox_email": "to@test.com",
    "message_count": "5",
    "mailbox_size_bytes": "75000",
    "mailbox_size_mb": "0.07",
    "quota_allocated_bytes": "1048576",
    "quota_allocated_mb": "1",
    "quota_used_bytes": "75000",
    "quota_used_mb": "0.07",
    "quota_remaining_bytes": "973576",
    "quota_remaining_mb": "0.93",
    "quota_percentage_used": "7.15",
    "is_over_quota": "false",
    "status": "WITHIN_LIMITS"
  }
}
```

## Python Example

Here's a complete Python example to launch the job and retrieve stats:

```python
import requests
import time
import json

AWX_URL = "https://your-awx-server"
TOKEN = "your_bearer_token"
TEMPLATE_ID = 10  # Your job template ID

headers = {
    "Authorization": f"Bearer {TOKEN}",
    "Content-Type": "application/json"
}

# Step 1: Launch job
launch_url = f"{AWX_URL}/api/v2/job_templates/{TEMPLATE_ID}/launch/"
launch_data = {
    "extra_vars": {
        "mailbox_email": "to@test.com"
    }
}

response = requests.post(launch_url, headers=headers, json=launch_data, verify=False)
job_id = response.json()["job"]
print(f"✅ Job launched: {job_id}")

# Step 2: Poll for completion
job_url = f"{AWX_URL}/api/v2/jobs/{job_id}/"
while True:
    response = requests.get(job_url, headers=headers, verify=False)
    job_data = response.json()
    status = job_data["status"]
    
    print(f"⏳ Job status: {status}")
    
    if status in ["successful", "failed", "error", "canceled"]:
        break
    
    time.sleep(5)  # Wait 5 seconds before polling again

# Step 3: Get artifacts
if status == "successful":
    artifacts = job_data.get("artifacts", {})
    print("\n📊 Quota Statistics:")
    print(json.dumps(artifacts, indent=2))
    
    # Check if over quota
    if artifacts.get("is_over_quota") == "true":
        print(f"\n⚠️  WARNING: Mailbox is OVER QUOTA!")
        print(f"   Used: {artifacts['quota_percentage_used']}%")
    else:
        print(f"\n✅ Mailbox is within limits")
        print(f"   Used: {artifacts['quota_percentage_used']}%")
        print(f"   Remaining: {artifacts['quota_remaining_mb']} MB")
else:
    print(f"\n❌ Job failed with status: {status}")
```

## JavaScript/Node.js Example

```javascript
const axios = require('axios');

const AWX_URL = 'https://your-awx-server';
const TOKEN = 'your_bearer_token';
const TEMPLATE_ID = 10;

const headers = {
  'Authorization': `Bearer ${TOKEN}`,
  'Content-Type': 'application/json'
};

async function runQuotaCheck(mailboxEmail) {
  try {
    // Step 1: Launch job
    const launchResponse = await axios.post(
      `${AWX_URL}/api/v2/job_templates/${TEMPLATE_ID}/launch/`,
      {
        extra_vars: {
          mailbox_email: mailboxEmail
        }
      },
      { headers, httpsAgent: new (require('https').Agent)({ rejectUnauthorized: false }) }
    );
    
    const jobId = launchResponse.data.job;
    console.log(`✅ Job launched: ${jobId}`);
    
    // Step 2: Poll for completion
    let status = 'pending';
    let artifacts = null;
    
    while (!['successful', 'failed', 'error', 'canceled'].includes(status)) {
      await new Promise(resolve => setTimeout(resolve, 5000)); // Wait 5 seconds
      
      const jobResponse = await axios.get(
        `${AWX_URL}/api/v2/jobs/${jobId}/`,
        { headers, httpsAgent: new (require('https').Agent)({ rejectUnauthorized: false }) }
      );
      
      status = jobResponse.data.status;
      artifacts = jobResponse.data.artifacts;
      console.log(`⏳ Job status: ${status}`);
    }
    
    // Step 3: Process results
    if (status === 'successful' && artifacts) {
      console.log('\n📊 Quota Statistics:');
      console.log(JSON.stringify(artifacts, null, 2));
      
      if (artifacts.is_over_quota === 'true') {
        console.log(`\n⚠️  WARNING: Mailbox is OVER QUOTA!`);
        console.log(`   Used: ${artifacts.quota_percentage_used}%`);
      } else {
        console.log(`\n✅ Mailbox is within limits`);
        console.log(`   Used: ${artifacts.quota_percentage_used}%`);
        console.log(`   Remaining: ${artifacts.quota_remaining_mb} MB`);
      }
      
      return artifacts;
    } else {
      throw new Error(`Job failed with status: ${status}`);
    }
    
  } catch (error) {
    console.error('Error:', error.message);
    throw error;
  }
}

// Usage
runQuotaCheck('to@test.com')
  .then(stats => console.log('Done!'))
  .catch(err => console.error('Failed:', err));
```

## Bash/cURL Example

```bash
#!/bin/bash

AWX_URL="https://your-awx-server"
TOKEN="your_bearer_token"
TEMPLATE_ID=10
MAILBOX_EMAIL="to@test.com"

# Step 1: Launch job
echo "🚀 Launching job template..."
LAUNCH_RESPONSE=$(curl -s -k -X POST \
  "${AWX_URL}/api/v2/job_templates/${TEMPLATE_ID}/launch/" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"extra_vars\": {\"mailbox_email\": \"${MAILBOX_EMAIL}\"}}")

JOB_ID=$(echo $LAUNCH_RESPONSE | jq -r '.job')
echo "✅ Job launched: ${JOB_ID}"

# Step 2: Poll for completion
while true; do
  JOB_STATUS=$(curl -s -k -X GET \
    "${AWX_URL}/api/v2/jobs/${JOB_ID}/" \
    -H "Authorization: Bearer ${TOKEN}" | jq -r '.status')
  
  echo "⏳ Job status: ${JOB_STATUS}"
  
  if [[ "$JOB_STATUS" == "successful" ]] || [[ "$JOB_STATUS" == "failed" ]]; then
    break
  fi
  
  sleep 5
done

# Step 3: Get artifacts
if [[ "$JOB_STATUS" == "successful" ]]; then
  echo ""
  echo "📊 Quota Statistics:"
  curl -s -k -X GET \
    "${AWX_URL}/api/v2/jobs/${JOB_ID}/" \
    -H "Authorization: Bearer ${TOKEN}" | jq '.artifacts'
else
  echo "❌ Job failed"
  exit 1
fi
```

## Testing Locally (Without AWX)

If you want to test the playbook locally and see the stats, run:

```bash
ansible-playbook playbooks/Get\ Mailbox\ Size\ and\ Compare\ Quota.yml -v
```

The stats will be displayed in the output under the `PLAY RECAP` section as custom stats.

## Integration with CI/CD

You can integrate this into your CI/CD pipeline to monitor mailbox quotas:

```yaml
# Example GitLab CI
check_mailbox_quota:
  stage: monitoring
  script:
    - |
      JOB_ID=$(curl -s -k -X POST \
        "${AWX_URL}/api/v2/job_templates/${TEMPLATE_ID}/launch/" \
        -H "Authorization: Bearer ${TOKEN}" \
        -H "Content-Type: application/json" \
        -d '{"extra_vars": {"mailbox_email": "to@test.com"}}' | jq -r '.job')
      
      # Wait for completion
      while true; do
        STATUS=$(curl -s -k "${AWX_URL}/api/v2/jobs/${JOB_ID}/" \
          -H "Authorization: Bearer ${TOKEN}" | jq -r '.status')
        [[ "$STATUS" == "successful" ]] || [[ "$STATUS" == "failed" ]] && break
        sleep 5
      done
      
      # Get stats and check quota
      IS_OVER_QUOTA=$(curl -s -k "${AWX_URL}/api/v2/jobs/${JOB_ID}/" \
        -H "Authorization: Bearer ${TOKEN}" | jq -r '.artifacts.is_over_quota')
      
      if [[ "$IS_OVER_QUOTA" == "true" ]]; then
        echo "ERROR: Mailbox is over quota!"
        exit 1
      fi
  only:
    - schedules
```

## Notes

- The `aggregate: no` parameter ensures stats are not aggregated across multiple hosts
- The `per_host: no` parameter ensures stats are returned at the play level, not per host
- All numeric values are returned as strings, so you may need to parse them
- Stats are available immediately after job completion via the `/api/v2/jobs/{id}/` endpoint
