# Get Top Processes Playbook - Reference Guide

## Overview

**Playbook:** `Get-Top-Processes.yml`

**Purpose:** Connect to a remote host via SSH and retrieve the top processes by memory and CPU usage, along with system resource information.

## Key Features

✅ **Top Memory Processes:** Retrieves top N processes sorted by memory usage  
✅ **Top CPU Processes:** Retrieves top N processes sorted by CPU usage  
✅ **System Info:** Captures memory usage, CPU load, and process count  
✅ **Flexible Count:** Configure how many top processes to retrieve  
✅ **Stats Export:** Returns all data to AWX/Tower job templates via `set_stats`  
✅ **Multi-host Support:** Can target specific hosts or groups  

## Parameters

| Parameter | Default | Required | Description |
|-----------|---------|----------|-------------|
| `target_host` | `localhost` | Yes | Target host or group to connect to |
| `top_count` | `10` | No | Number of top processes to retrieve (1-100) |

## Usage Examples

### 1. Get Top Processes from Specific Host

```bash
ansible-playbook playbooks/Get-Top-Processes.yml -e "target_host=webserver"
```

### 2. Get Top 5 Processes

```bash
ansible-playbook playbooks/Get-Top-Processes.yml \
  -e "target_host=dbserver" \
  -e "top_count=5"
```

### 3. Get Top Processes from All Hosts

```bash
ansible-playbook playbooks/Get-Top-Processes.yml -e "target_host=all"
```

### 4. Get Top 20 Processes from Production Servers

```bash
ansible-playbook playbooks/Get-Top-Processes.yml \
  -e "target_host=production" \
  -e "top_count=20"
```

## AWX/Tower Integration

### Launch Job Template

```json
{
  "target_host": "webserver",
  "top_count": 10
}
```

### Retrieve Stats from Job

After job completion, retrieve artifacts:

```bash
curl -X GET https://awx-server/api/v2/jobs/{job_id}/ \
  -H "Authorization: Bearer TOKEN"
```

**Response includes:**

```json
{
  "artifacts": {
    "target_host": "webserver01",
    "os": "Ubuntu 22.04",
    "cpu_count": "4",
    "load_1min": "0.52",
    "load_5min": "0.48",
    "load_15min": "0.45",
    "memory_total_mb": "8192",
    "memory_used_mb": "4096",
    "memory_free_mb": "4096",
    "memory_usage_percent": "50.00",
    "total_process_count": "187",
    "top_memory_processes": "[{\"pid\":\"1234\",\"user\":\"mysql\",\"cpu\":\"2.5\",\"mem\":\"15.2\",\"command\":\"mysqld\"}...]",
    "top_cpu_processes": "[{\"pid\":\"5678\",\"user\":\"www-data\",\"cpu\":\"45.2\",\"mem\":\"3.1\",\"command\":\"php-fpm\"}...]",
    "top_memory_process_1": "mysqld",
    "top_memory_process_1_usage": "15.2",
    "top_cpu_process_1": "php-fpm",
    "top_cpu_process_1_usage": "45.2",
    "timestamp": "2026-01-28T14:30:00Z",
    "status": "SUCCESS"
  }
}
```

## Stats Exported

### System Information

| Stat Key | Description | Example |
|----------|-------------|---------|
| `target_host` | Hostname of the target | `webserver01` |
| `os` | Operating system and version | `Ubuntu 22.04` |
| `cpu_count` | Number of CPU cores | `4` |
| `load_1min` | 1-minute load average | `0.52` |
| `load_5min` | 5-minute load average | `0.48` |
| `load_15min` | 15-minute load average | `0.45` |
| `memory_total_mb` | Total memory in MB | `8192` |
| `memory_used_mb` | Used memory in MB | `4096` |
| `memory_free_mb` | Free memory in MB | `4096` |
| `memory_usage_percent` | Memory usage percentage | `50.00` |
| `total_process_count` | Total running processes | `187` |

### Process Information

| Stat Key | Description | Example |
|----------|-------------|---------|
| `top_memory_processes` | JSON array of top processes by memory | `[{"pid":"1234",...}]` |
| `top_cpu_processes` | JSON array of top processes by CPU | `[{"pid":"5678",...}]` |
| `top_memory_process_1` | Command of top memory consumer | `mysqld` |
| `top_memory_process_1_usage` | Memory % of top consumer | `15.2` |
| `top_cpu_process_1` | Command of top CPU consumer | `php-fpm` |
| `top_cpu_process_1_usage` | CPU % of top consumer | `45.2` |

### Process Object Structure

Each process in the arrays contains:

```json
{
  "pid": "1234",
  "user": "mysql",
  "cpu": "2.5",
  "mem": "15.2",
  "vsz": "1234567",
  "rss": "524288",
  "command": "mysqld"
}
```

| Field | Description |
|-------|-------------|
| `pid` | Process ID |
| `user` | User running the process |
| `cpu` | CPU usage percentage |
| `mem` | Memory usage percentage |
| `vsz` | Virtual memory size (KB) |
| `rss` | Resident set size (KB) |
| `command` | Command name |

## Python Example - AWX Integration

```python
import requests
import time
import json

AWX_URL = "https://awx-server"
TOKEN = "your_token"
TEMPLATE_ID = 25

headers = {"Authorization": f"Bearer {TOKEN}"}

# Launch job
response = requests.post(
    f"{AWX_URL}/api/v2/job_templates/{TEMPLATE_ID}/launch/",
    headers=headers,
    json={
        "extra_vars": {
            "target_host": "webserver",
            "top_count": 10
        }
    }
)

job_id = response.json()["job"]
print(f"✅ Job launched: {job_id}")

# Poll for completion
while True:
    job = requests.get(f"{AWX_URL}/api/v2/jobs/{job_id}/", headers=headers).json()
    status = job["status"]
    print(f"⏳ Job status: {status}")
    
    if status in ["successful", "failed"]:
        break
    time.sleep(3)

# Get results
if status == "successful":
    artifacts = job.get("artifacts", {})
    
    print(f"\n📊 System Information:")
    print(f"   Host: {artifacts['target_host']}")
    print(f"   OS: {artifacts['os']}")
    print(f"   CPU Cores: {artifacts['cpu_count']}")
    print(f"   Load: {artifacts['load_1min']} / {artifacts['load_5min']} / {artifacts['load_15min']}")
    print(f"   Memory: {artifacts['memory_used_mb']} MB / {artifacts['memory_total_mb']} MB ({artifacts['memory_usage_percent']}%)")
    
    print(f"\n🔥 Top Memory Consumer: {artifacts['top_memory_process_1']} ({artifacts['top_memory_process_1_usage']}%)")
    print(f"⚡ Top CPU Consumer: {artifacts['top_cpu_process_1']} ({artifacts['top_cpu_process_1_usage']}%)")
    
    # Parse full process lists
    top_memory = json.loads(artifacts['top_memory_processes'])
    top_cpu = json.loads(artifacts['top_cpu_processes'])
    
    print(f"\n📋 Top 10 by Memory:")
    for i, proc in enumerate(top_memory[:10], 1):
        print(f"   {i}. {proc['command']} (PID: {proc['pid']}) - {proc['mem']}% MEM, {proc['cpu']}% CPU")
    
    print(f"\n📋 Top 10 by CPU:")
    for i, proc in enumerate(top_cpu[:10], 1):
        print(f"   {i}. {proc['command']} (PID: {proc['pid']}) - {proc['cpu']}% CPU, {proc['mem']}% MEM")
else:
    print(f"\n❌ Job failed")
```

## Bash/cURL Example

```bash
#!/bin/bash

AWX_URL="https://awx-server"
TOKEN="your_token"
TEMPLATE_ID=25
TARGET_HOST="webserver"

# Launch job
echo "🚀 Getting top processes from ${TARGET_HOST}..."
LAUNCH_RESPONSE=$(curl -s -k -X POST \
  "${AWX_URL}/api/v2/job_templates/${TEMPLATE_ID}/launch/" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"extra_vars\": {\"target_host\": \"${TARGET_HOST}\", \"top_count\": 10}}")

JOB_ID=$(echo $LAUNCH_RESPONSE | jq -r '.job')
echo "✅ Job launched: ${JOB_ID}"

# Poll for completion
while true; do
  STATUS=$(curl -s -k "${AWX_URL}/api/v2/jobs/${JOB_ID}/" \
    -H "Authorization: Bearer ${TOKEN}" | jq -r '.status')
  echo "⏳ Status: ${STATUS}"
  [[ "$STATUS" == "successful" ]] || [[ "$STATUS" == "failed" ]] && break
  sleep 3
done

# Get results
if [[ "$STATUS" == "successful" ]]; then
  echo ""
  echo "📊 System & Process Stats:"
  curl -s -k "${AWX_URL}/api/v2/jobs/${JOB_ID}/" \
    -H "Authorization: Bearer ${TOKEN}" | jq '.artifacts | {
      host: .target_host,
      os: .os,
      cpu_count: .cpu_count,
      load: "\(.load_1min) / \(.load_5min) / \(.load_15min)",
      memory: "\(.memory_used_mb)MB / \(.memory_total_mb)MB (\(.memory_usage_percent)%)",
      top_memory_process: "\(.top_memory_process_1) (\(.top_memory_process_1_usage)%)",
      top_cpu_process: "\(.top_cpu_process_1) (\(.top_cpu_process_1_usage)%)"
    }'
else
  echo "❌ Job failed"
  exit 1
fi
```

## Sample Output

When running the playbook, you'll see output like:

```
TASK [Display top processes by MEMORY] *****************************************
ok: [webserver] => {
    "msg": [
        "==================================",
        "  TOP 10 PROCESSES BY MEMORY",
        "==================================",
        "1. PID: 1234 | User: mysql | MEM: 15.2% | CPU: 2.5% | mysqld",
        "2. PID: 2345 | User: java | MEM: 12.8% | CPU: 8.3% | java",
        "3. PID: 3456 | User: redis | MEM: 8.5% | CPU: 1.2% | redis-server",
        ...
    ]
}

TASK [Display system summary] **************************************************
ok: [webserver] => {
    "msg": [
        "==================================",
        "       SYSTEM SUMMARY",
        "==================================",
        "Host: webserver",
        "OS: Ubuntu 22.04",
        "CPU Cores: 4",
        "Load Average: 0.52 / 0.48 / 0.45",
        "Memory: 4096 MB used / 8192 MB total (50.00%)",
        "Total Processes: 187",
        "=================================="
    ]
}
```

## Use Cases

### 1. Performance Monitoring

Check which processes are consuming the most resources:

```bash
ansible-playbook playbooks/Get-Top-Processes.yml \
  -e "target_host=production_servers"
```

### 2. Troubleshooting High Load

Identify resource-hungry processes during incidents:

```bash
ansible-playbook playbooks/Get-Top-Processes.yml \
  -e "target_host=slow_server" \
  -e "top_count=20"
```

### 3. Capacity Planning

Collect resource usage data across infrastructure:

```bash
ansible-playbook playbooks/Get-Top-Processes.yml \
  -e "target_host=all"
```

### 4. Scheduled Health Checks

Run via AWX/Tower scheduled job to collect daily metrics.

### 5. Pre/Post Deployment Comparison

Compare process usage before and after deployments.

## Important Notes

⚠️ **Linux Only:** Uses `ps aux` and `/proc/loadavg` (Linux-specific)  
⚠️ **SSH Access Required:** Needs SSH connectivity to target hosts  
⚠️ **No Root Required:** Runs without elevated privileges  
⚠️ **Read-Only:** Does not modify any system configuration  

## Troubleshooting

### "Host unreachable"

- Verify SSH connectivity: `ssh user@target_host`
- Check inventory file has correct hostname
- Verify SSH keys are configured

### "Permission denied"

- Check SSH user has access to target host
- Verify `authorized_keys` on target

### Empty process list

- Verify `ps` command is available
- Check if target is a minimal container (may not have full `ps`)

### Load average seems high

- Compare with CPU count (load of 4.0 on 4-core system = 100% utilized)
- High load without high CPU may indicate I/O wait

## Related Playbooks

- **Kill Process** - Terminate problematic processes
- **System Health Check** - Comprehensive system health monitoring
- **Memory Cleanup** - Clear caches and free memory

---

**Last Updated:** 2026-01-28  
**Version:** 1.0 - Initial release
