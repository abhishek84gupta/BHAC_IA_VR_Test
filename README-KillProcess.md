# Kill Process Playbook - Reference Guide

## Overview

**Playbook:** `09-kill-process.yml`

**Purpose:** Identify and terminate processes by name or PID on remote servers with verification and comprehensive reporting.

## Key Features

✅ **Flexible Targeting:** Kill by process name or specific PID  
✅ **Configurable Signals:** TERM, KILL, HUP, INT, QUIT, USR1, USR2  
✅ **Graceful Termination:** Attempts graceful shutdown before force kill  
✅ **Verification:** Confirms process termination  
✅ **Multi-host Support:** Can target specific hosts or all hosts  
✅ **Stats Export:** Returns results to AWX/Tower job templates  

## Required Parameters

**At least ONE of the following must be provided:**
- `process_name` - Name of the process to kill (e.g., "nginx", "httpd")
- `process_pid` - Specific PID to terminate (e.g., "12345")

## Optional Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `target_host` | `all` | Target host or group (e.g., "webserver", "dbserver") |
| `signal` | `TERM` | Signal to send (TERM, KILL, HUP, INT, QUIT, USR1, USR2) |
| `kill_all_matches` | `false` | Kill all matching processes or just first |
| `verify_kill` | `true` | Verify process was terminated |

## Usage Examples

### 1. Kill Process by Name (Default Host)

```bash
ansible-playbook playbooks/09-kill-process.yml -e "process_name=nginx"
```

### 2. Kill Process by PID

```bash
ansible-playbook playbooks/09-kill-process.yml -e "process_pid=12345"
```

### 3. Target Specific Host

```bash
ansible-playbook playbooks/09-kill-process.yml \
  -e "target_host=webserver" \
  -e "process_name=httpd"
```

### 4. Force Kill (SIGKILL)

```bash
ansible-playbook playbooks/09-kill-process.yml \
  -e "process_name=stuck_process" \
  -e "signal=KILL"
```

### 5. Kill on Multiple Hosts

```bash
ansible-playbook playbooks/09-kill-process.yml \
  -e "target_host=webservers" \
  -e "process_name=apache2"
```

### 6. Send Custom Signal

```bash
ansible-playbook playbooks/09-kill-process.yml \
  -e "process_name=myapp" \
  -e "signal=HUP"
```

## AWX/Tower Integration

### Launch Job Template

```json
{
  "target_host": "webserver",
  "process_name": "nginx",
  "signal": "TERM"
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
    "process_name": "nginx",
    "process_pid": "N/A",
    "signal_sent": "TERM",
    "pids_processed": "2",
    "pids_list": "1234, 5678",
    "terminated_count": "2",
    "still_running_count": "0",
    "status": "SUCCESS",
    "all_terminated": "true"
  }
}
```

## Process Flow

1. **Configuration** - Set variables from extra_vars or use defaults
2. **Validation** - Ensure required parameters provided and signal is valid
3. **Discovery** - Find processes matching name or verify PID exists
4. **Termination** - Send signal (graceful first, then force if needed)
5. **Verification** - Confirm process termination
6. **Reporting** - Display summary and export stats

## Signal Types

| Signal | Description | Use Case |
|--------|-------------|----------|
| `TERM` | Graceful termination (default) | Normal shutdown |
| `KILL` | Force kill (immediate) | Unresponsive processes |
| `HUP` | Hang up | Reload configuration |
| `INT` | Interrupt | Similar to Ctrl+C |
| `QUIT` | Quit with core dump | Debug crashed process |
| `USR1` | User-defined signal 1 | Application-specific |
| `USR2` | User-defined signal 2 | Application-specific |

## Return Codes / Status

| Status | Description |
|--------|-------------|
| `SUCCESS` | All processes terminated successfully |
| `PARTIAL_FAILURE` | Some processes still running |
| `NO_PROCESSES_FOUND` | No matching processes found |

## Python Example - AWX Integration

```python
import requests
import time

AWX_URL = "https://awx-server"
TOKEN = "your_token"
TEMPLATE_ID = 15

headers = {"Authorization": f"Bearer {TOKEN}"}

# Launch job
response = requests.post(
    f"{AWX_URL}/api/v2/job_templates/{TEMPLATE_ID}/launch/",
    headers=headers,
    json={
        "extra_vars": {
            "target_host": "webserver",
            "process_name": "nginx",
            "signal": "TERM"
        }
    }
)

job_id = response.json()["job"]
print(f"Job launched: {job_id}")

# Poll for completion
while True:
    job = requests.get(f"{AWX_URL}/api/v2/jobs/{job_id}/", headers=headers).json()
    if job["status"] in ["successful", "failed"]:
        break
    time.sleep(3)

# Get results
artifacts = job.get("artifacts", {})
print(f"Status: {artifacts['status']}")
print(f"Terminated: {artifacts['terminated_count']}")
print(f"Still Running: {artifacts['still_running_count']}")

if artifacts["all_terminated"] == "true":
    print("✅ All processes terminated successfully")
else:
    print("⚠️ Some processes are still running")
```

## Important Notes

⚠️ **Requires elevated privileges** - Playbook uses `become: yes` to run as root  
⚠️ **Careful with wildcards** - Process name matches can be broad  
⚠️ **KILL signal** - Use with caution, doesn't allow cleanup  
⚠️ **Verify before running** - Review found processes before killing  

## Troubleshooting

### "No processes found"
- Check process name spelling
- Verify process is running: `ps aux | grep process_name`
- Check target host is correct

### "Process still running after kill"
- Try with `signal=KILL` for force termination
- Check if process is zombieor has parent process keeping it alive
- Verify user has permission to kill the process

### "Permission denied"
- Ensure `become: yes` is working
- Check SSH user has sudo privileges
- Verify process isn't owned by root (if not using sudo)

## Examples by Use Case

### Restart Web Server
```bash
# Gracefully stop nginx
ansible-playbook playbooks/09-kill-process.yml \
  -e "target_host=webservers" \
  -e "process_name=nginx" \
  -e "signal=TERM"

# Then start it again (separate playbook)
```

### Force Kill Stuck Process
```bash
ansible-playbook playbooks/09-kill-process.yml \
  -e "process_pid=12345" \
  -e "signal=KILL"
```

### Reload Application Configuration
```bash
ansible-playbook playbooks/09-kill-process.yml \
  -e "process_name=myapp" \
  -e "signal=HUP"
```

### Clean Up Test Processes
```bash
ansible-playbook playbooks/09-kill-process.yml \
  -e "process_name=test_runner" \
  -e "kill_all_matches=yes"
```

## Related Playbooks

- **Service Management** - Use for managed services (systemd, etc.)
- **Health Checks** - Verify processes after termination
- **Log Collection** - Gather logs before killing process

---

**Last Updated:** 2026-01-28  
**Version:** 2.0 - Added extra_vars support and stats export
