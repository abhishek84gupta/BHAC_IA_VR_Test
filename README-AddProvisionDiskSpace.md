# Add/Provision Disk Space Playbook - Reference Guide

## Overview

**Playbook:** `Add-Provision-Disk-Space.yml`

**Purpose:** Dummy action to simulate disk space provisioning. This playbook does NOT make actual changes - it only prints messages to simulate the provisioning workflow.

## Important Notice

⚠️ **THIS IS A SIMULATION PLAYBOOK** ⚠️

This playbook performs NO actual disk provisioning. It:
- Prints messages simulating disk provisioning steps
- Shows current disk usage
- Returns stats to job template caller
- Does NOT create volumes, format disks, or mount filesystems

Use this for:
- Testing job template workflows
- Demonstrating provisioning processes
- Template for building actual provisioning logic

## Key Features

✅ **Simulation Mode:** Safe dummy action with no system changes  
✅ **Dynamic Host:** Accept hostname via extra_vars (no inventory needed)  
✅ **Configurable:** Disk size, mount point, filesystem type  
✅ **Stats Export:** Returns provisioning details to AWX/Tower  
✅ **Current Usage:** Shows actual disk usage on target host  

## Parameters

| Parameter | Default | Required | Description |
|-----------|---------|----------|-------------|
| `target_host` | - | **Yes** | Target host to "provision" disk on |
| `disk_size_gb` | `50` | No | Disk size to provision in GB (1-10000) |
| `mount_point` | `/data` | No | Mount point for the new disk |
| `filesystem_type` | `ext4` | No | Filesystem type (ext4, xfs, etc.) |
| `ansible_user` | Current user | No | SSH username |
| `ansible_port` | `22` | No | SSH port |

## Usage Examples

### 1. Simulate 50 GB Disk Provisioning

```bash
ansible-playbook playbooks/Add-Provision-Disk-Space.yml \
  -e "target_host=webserver.example.com" \
  -e "ansible_user=admin" \
  -k
```

### 2. Simulate 100 GB Disk with Custom Mount Point

```bash
ansible-playbook playbooks/Add-Provision-Disk-Space.yml \
  -e "target_host=192.168.1.100" \
  -e "disk_size_gb=100" \
  -e "mount_point=/mnt/storage" \
  -e "ansible_user=root" \
  -k
```

### 3. Simulate XFS Filesystem

```bash
ansible-playbook playbooks/Add-Provision-Disk-Space.yml \
  -e "target_host=dbserver.example.com" \
  -e "disk_size_gb=200" \
  -e "mount_point=/database" \
  -e "filesystem_type=xfs"
```

### 4. Run on Localhost

```bash
ansible-playbook playbooks/Add-Provision-Disk-Space.yml \
  -e "target_host=localhost" \
  -e "disk_size_gb=20"
```

## AWX/Tower Integration

### Launch Job Template

```json
{
  "target_host": "webserver.example.com",
  "disk_size_gb": 50,
  "mount_point": "/data",
  "filesystem_type": "ext4",
  "ansible_user": "admin"
}
```

### Retrieve Stats from Job

After job completion:

```bash
curl -X GET https://awx-server/api/v2/jobs/{job_id}/ \
  -H "Authorization: Bearer TOKEN"
```

**Response includes:**

```json
{
  "artifacts": {
    "target_host": "webserver01",
    "disk_size_requested_gb": "50",
    "mount_point": "/data",
    "filesystem_type": "ext4",
    "provisioning_status": "SIMULATED",
    "action_type": "DUMMY",
    "message": "Disk provisioning simulated successfully. No actual changes were made.",
    "timestamp": "2026-01-28T15:00:00Z",
    "current_disk_usage": "[\"Filesystem      Size  Used Avail Use% Mounted on\", ...]",
    "status": "SUCCESS",
    "warning": "This was a simulation only - no actual disk was provisioned"
  }
}
```

## Sample Output

```
TASK [Display provisioning configuration] **************************************
ok: [webserver.example.com] => {
    "msg": [
        "==================================",
        "  DISK PROVISIONING REQUEST",
        "==================================",
        "🎯 Target Host: webserver.example.com",
        "💾 Disk Size: 50 GB",
        "📁 Mount Point: /data",
        "🗂️  Filesystem Type: ext4",
        "=================================="
    ]
}

TASK [Simulate disk provisioning (DUMMY ACTION)] *******************************
ok: [webserver.example.com] => {
    "msg": [
        "⚙️  Simulating disk provisioning...",
        "",
        "Step 1: ✅ Allocating 50 GB of disk space from storage pool",
        "Step 2: ✅ Creating new volume: /dev/sdb1",
        "Step 3: ✅ Formatting volume with ext4 filesystem",
        "Step 4: ✅ Creating mount point: /data",
        "Step 5: ✅ Mounting volume to /data",
        "Step 6: ✅ Adding entry to /etc/fstab for persistence",
        "Step 7: ✅ Setting appropriate permissions",
        "",
        "✨ Disk provisioning completed successfully!",
        "",
        "⚠️  NOTE: This is a DUMMY action. No actual changes were made."
    ]
}
```

## Converting to Real Provisioning

To implement actual disk provisioning, you would need to:

### For Cloud Environments

**AWS:**
```yaml
- name: Create and attach EBS volume
  amazon.aws.ec2_vol:
    instance: "{{ instance_id }}"
    volume_size: "{{ disk_size_gb }}"
    volume_type: gp3
    device_name: /dev/sdf
    state: present
```

**Azure:**
```yaml
- name: Create managed disk
  azure.azcollection.azure_rm_manageddisk:
    name: "data-disk-{{ disk_size_gb }}gb"
    disk_size_gb: "{{ disk_size_gb }}"
    managed_by: "{{ vm_name }}"
```

**GCP:**
```yaml
- name: Create persistent disk
  google.cloud.gcp_compute_disk:
    name: "data-disk-{{ disk_size_gb }}gb"
    size_gb: "{{ disk_size_gb }}"
    zone: us-central1-a
```

### For On-Premise/LVM

```yaml
- name: Create LVM logical volume
  lvol:
    vg: vg_data
    lv: lv_data
    size: "{{ disk_size_gb }}g"
    state: present

- name: Format filesystem
  filesystem:
    fstype: "{{ filesystem_type }}"
    dev: "/dev/vg_data/lv_data"

- name: Create mount point
  file:
    path: "{{ mount_point }}"
    state: directory
    mode: '0755'

- name: Mount filesystem
  mount:
    path: "{{ mount_point }}"
    src: "/dev/vg_data/lv_data"
    fstype: "{{ filesystem_type }}"
    state: mounted
```

## Python Example - AWX Integration

```python
import requests
import time

AWX_URL = "https://awx-server"
TOKEN = "your_token"
TEMPLATE_ID = 30

headers = {"Authorization": f"Bearer {TOKEN}"}

# Launch disk provisioning job
response = requests.post(
    f"{AWX_URL}/api/v2/job_templates/{TEMPLATE_ID}/launch/",
    headers=headers,
    json={
        "extra_vars": {
            "target_host": "webserver.example.com",
            "disk_size_gb": 100,
            "mount_point": "/data",
            "filesystem_type": "ext4",
            "ansible_user": "admin"
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
    print(f"\n✅ Disk Provisioning Results:")
    print(f"   Host: {artifacts['target_host']}")
    print(f"   Size Requested: {artifacts['disk_size_requested_gb']} GB")
    print(f"   Mount Point: {artifacts['mount_point']}")
    print(f"   Status: {artifacts['provisioning_status']}")
    print(f"   ⚠️  {artifacts['warning']}")
else:
    print(f"\n❌ Job failed")
```

## Use Cases

### 1. Workflow Testing
Test AWX/Tower job template workflows without making actual changes.

### 2. Approval Process Demo
Demonstrate approval workflows for storage requests.

### 3. Template Development
Use as a starting point to build actual provisioning logic.

### 4. Training
Show what disk provisioning workflow would look like.

## Important Notes

⚠️ **No Real Changes:** This playbook makes NO actual system changes  
⚠️ **Simulation Only:** All provisioning steps are simulated messages  
⚠️ **Root Access:** Uses `become: yes` to show what real provisioning needs  
⚠️ **Stats Export:** Returns simulated provisioning details  

## Extending to Real Provisioning

When ready to implement real provisioning:

1. **Replace the dummy task** with actual cloud provider module (AWS, Azure, GCP) or storage commands (LVM, partition tools)
2. **Add pre-checks** to verify available storage capacity
3. **Add rollback logic** in case of failures
4. **Add verification** to confirm disk is properly mounted
5. **Update fstab** for persistence across reboots
6. **Set permissions** on the mounted filesystem

## Related Playbooks

- **Disk Usage Analysis** - Check current disk usage before provisioning
- **Extend Disk** - Expand existing filesystems
- **Mount Management** - Manage mount points

---

**Last Updated:** 2026-01-28  
**Version:** 1.0 - Initial dummy implementation
