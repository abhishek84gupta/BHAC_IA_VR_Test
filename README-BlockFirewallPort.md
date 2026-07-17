# Block Firewall Port Playbook - Reference Guide

## Overview

**Playbook:** `03-block-firewall-port.yml`

**Purpose:** Block a specific port on remote servers using the appropriate firewall method (firewalld, UFW, or iptables).

## Key Features

✅ **Multi-Platform Support:** Automatically detects and uses firewalld, UFW, or iptables  
✅ **Flexible Configuration:** Accept port and hostname via extra_vars  
✅ **Protocol Support:** TCP or UDP  
✅ **Verification:** Confirms port blocking after application  
✅ **Stats Export:** Returns firewall status to AWX/Tower job templates  
✅ **Multi-host Support:** Can target specific hosts or all hosts  

## Parameters

| Parameter | Default | Required | Description |
|-----------|---------|----------|-------------|
| `target_host` | `all` | No | Target host or group (e.g., "webserver", "dbserver") |
| `port_to_block` | `8080` | Yes | Port number to block (1-65535) |
| `protocol` | `tcp` | No | Protocol to block (tcp or udp) |

## Usage Examples

### 1. Block Default Port (8080) on All Hosts

```bash
ansible-playbook playbooks/03-block-firewall-port.yml
```

### 2. Block Specific Port

```bash
ansible-playbook playbooks/03-block-firewall-port.yml -e "port_to_block=443"
```

### 3. Block Port on Specific Host

```bash
ansible-playbook playbooks/03-block-firewall-port.yml \
  -e "target_host=webserver" \
  -e "port_to_block=8080"
```

### 4. Block UDP Port

```bash
ansible-playbook playbooks/03-block-firewall-port.yml \
  -e "port_to_block=53" \
  -e "protocol=udp"
```

### 5. Block MySQL Port on Database Servers

```bash
ansible-playbook playbooks/03-block-firewall-port.yml \
  -e "target_host=dbservers" \
  -e "port_to_block=3306"
```

### 6. Block HTTPS on Web Servers

```bash
ansible-playbook playbooks/03-block-firewall-port.yml \
  -e "target_host=webservers" \
  -e "port_to_block=443" \
  -e "protocol=tcp"
```

## AWX/Tower Integration

### Launch Job Template

```json
{
  "target_host": "webserver",
  "port_to_block": "8080",
  "protocol": "tcp"
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
    "port_blocked": "8080",
    "protocol": "tcp",
    "firewall_method": "firewalld",
    "action_taken": "true",
    "verified": "verified",
    "status": "SUCCESS",
    "os_family": "RedHat",
    "timestamp": "2026-01-28T12:34:56Z"
  }
}
```

## Firewall Methods

The playbook automatically detects and uses the appropriate firewall method:

### 1. Firewalld (RHEL/CentOS/Fedora)

**Detection:** Checks if firewalld service is active  
**Actions:**
- Removes port from allowed list (if present)
- Adds rich rule to explicitly block port
- Reloads firewalld to apply changes

**Commands used:**
```bash
firewall-cmd --remove-port=8080/tcp --permanent
firewall-cmd --add-rich-rule='rule family="ipv4" port port="8080" protocol="tcp" reject' --permanent
firewall-cmd --reload
```

### 2. UFW (Ubuntu/Debian)

**Detection:** Checks if `ufw` command is available  
**Actions:**
- Adds deny rule for specified port

**Commands used:**
```bash
ufw deny 8080/tcp
```

### 3. iptables (Fallback)

**Detection:** Used if neither firewalld nor UFW is available  
**Actions:**
- Adds DROP rule to INPUT chain
- Saves iptables rules

**Commands used:**
```bash
iptables -A INPUT -p tcp --dport 8080 -j DROP -m comment --comment "Block port 8080 via Ansible"
iptables-save > /etc/iptables/rules.v4
```

## Process Flow

1. **Configuration** - Set variables from extra_vars or use defaults
2. **Validation** - Verify port number (1-65535) and protocol (tcp/udp)
3. **Detection** - Identify available firewall method
4. **Blocking** - Apply firewall rule using detected method
5. **Verification** - Confirm port is blocked
6. **Reporting** - Display summary and export stats

## Return Status

| Status | Description |
|--------|-------------|
| `SUCCESS` | Port blocked successfully |
| `NO_ACTION` | No action taken (firewall not available) |

## Python Example - AWX Integration

```python
import requests
import time
import json

AWX_URL = "https://awx-server"
TOKEN = "your_token"
TEMPLATE_ID = 20

headers = {"Authorization": f"Bearer {TOKEN}"}

# Launch job to block port
response = requests.post(
    f"{AWX_URL}/api/v2/job_templates/{TEMPLATE_ID}/launch/",
    headers=headers,
    json={
        "extra_vars": {
            "target_host": "webserver",
            "port_to_block": "8080",
            "protocol": "tcp"
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
    print(f"\n📊 Firewall Blocking Results:")
    print(json.dumps(artifacts, indent=2))
    
    if artifacts["status"] == "SUCCESS":
        print(f"\n✅ Port {artifacts['port_blocked']} blocked successfully")
        print(f"   Method: {artifacts['firewall_method']}")
        print(f"   Verified: {artifacts['verified']}")
    else:
        print(f"\n⚠️ No action taken on {artifacts['target_host']}")
else:
    print(f"\n❌ Job failed")
```

## Bash/cURL Example

```bash
#!/bin/bash

AWX_URL="https://awx-server"
TOKEN="your_token"
TEMPLATE_ID=20
TARGET_HOST="webserver"
PORT="8080"

# Launch job
echo "🚀 Blocking port ${PORT} on ${TARGET_HOST}..."
LAUNCH_RESPONSE=$(curl -s -k -X POST \
  "${AWX_URL}/api/v2/job_templates/${TEMPLATE_ID}/launch/" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"extra_vars\": {\"target_host\": \"${TARGET_HOST}\", \"port_to_block\": \"${PORT}\"}}")

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
  echo "📊 Firewall Status:"
  curl -s -k "${AWX_URL}/api/v2/jobs/${JOB_ID}/" \
    -H "Authorization: Bearer ${TOKEN}" | jq '.artifacts'
else
  echo "❌ Job failed"
  exit 1
fi
```

## Common Port Numbers Reference

| Port | Service | Protocol | Description |
|------|---------|----------|-------------|
| 22 | SSH | TCP | Secure Shell (⚠️ Be careful!) |
| 80 | HTTP | TCP | Web traffic |
| 443 | HTTPS | TCP | Secure web traffic |
| 3306 | MySQL | TCP | MySQL database |
| 5432 | PostgreSQL | TCP | PostgreSQL database |
| 6379 | Redis | TCP | Redis cache |
| 8080 | HTTP Alt | TCP | Alternative HTTP port |
| 9200 | Elasticsearch | TCP | Elasticsearch API |
| 27017 | MongoDB | TCP | MongoDB database |

## Important Notes

⚠️ **Be Careful with SSH (Port 22):** Blocking SSH can lock you out of the server  
⚠️ **Root/Sudo Required:** Playbook uses `become: yes` for elevated privileges  
⚠️ **Persistence:** Changes are made permanent (survive reboots)  
⚠️ **Testing:** Test on non-production systems first  
⚠️ **Verification:** Always verify port is actually blocked  

## Troubleshooting

### Port Still Accessible

**Check firewall status:**
```bash
# firewalld
sudo firewall-cmd --list-all

# UFW
sudo ufw status verbose

# iptables
sudo iptables -L INPUT -n -v
```

**Possible causes:**
- Firewall service not running
- Rules not saved/persisted
- Docker or other services bypassing firewall
- Application bound to 0.0.0.0 (all interfaces)

### "No firewall method available"

**Install firewall:**
```bash
# RHEL/CentOS
sudo yum install firewalld
sudo systemctl enable --now firewalld

# Ubuntu/Debian
sudo apt install ufw
sudo ufw enable

# iptables
sudo apt install iptables-persistent
```

### Permission Denied

- Ensure SSH user has sudo privileges
- Verify `become: yes` is working
- Check `/etc/sudoers` configuration

### Changes Don't Persist After Reboot

- Firewalld: Use `--permanent` flag (playbook already does this)
- UFW: Ensure UFW is enabled (`sudo ufw enable`)
- iptables: Save rules with `iptables-save` (playbook already does this)

## Security Best Practices

1. **Document Changes:** Keep track of blocked ports
2. **Review Regularly:** Audit firewall rules periodically
3. **Test First:** Use staging environment before production
4. **Monitor Impact:** Watch for service disruptions
5. **Backup Rules:** Save firewall configs before changes
6. **Use Allowlist:** Consider allowing specific ports instead of blocking

## Unblocking Ports

To unblock a port, you'll need a separate playbook or manual commands:

**firewalld:**
```bash
sudo firewall-cmd --remove-rich-rule='rule family="ipv4" port port="8080" protocol="tcp" reject' --permanent
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
```

**UFW:**
```bash
sudo ufw delete deny 8080/tcp
sudo ufw allow 8080/tcp
```

**iptables:**
```bash
sudo iptables -D INPUT -p tcp --dport 8080 -j DROP
sudo iptables-save > /etc/iptables/rules.v4
```

## Related Playbooks

- **Unblock Port** - Reverse operation (create separate playbook)
- **List Firewall Rules** - Audit current rules
- **Security Hardening** - Comprehensive security configuration

---

**Last Updated:** 2026-01-28  
**Version:** 2.0 - Added extra_vars support and stats export
