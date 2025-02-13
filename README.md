# Proxmox VM Disk Resizer
## Overview

The Proxmox VM Disk Resizer is an automated service that dynamically increases the disk size of virtual machines in a Proxmox Virtual Environment (PVE). It runs as a webhook-driven WEBrick HTTP server, listens for requests, and adjusts disk sizes accordingly. The script also ensures that the filesystem is expanded after the resize operation.

## Features

✅ Automated Disk Expansion – Increases VM disk size by 10% dynamically.

✅ Filesystem Resize Support – Expands XFS filesystems after resizing.

✅ Proxmox API Integration – Uses authenticated API calls to adjust VM storage.

✅ Slack Notifications – Sends success/failure messages via Slack Webhooks.

✅ Secure Credentials Handling – Reads credentials from Vault storage.


## How It Works
The script receives a webhook request with VM metadata (IP, mount point, device, etc.).
It authenticates with Proxmox API to fetch disk information.
Identifies the SCSI disk associated with the VM and calculates the new size.
Calls the Proxmox resize API to increase the disk size.
Expands the filesystem using ssh and XFS growfs.
Sends status notifications to Slack.

## Configuration
The script reads Proxmox and VM ssh key credentials from /vault/secrets/config.
Instead using Hashicorp Vault You can do this with environment variables or kubernetes secret.
You should setup ssh key based access to the VM's.
Only specific predefined mount points (by my example /media/data, /var/lib/mongodb) are eligible for resizing.
Uses HTTParty for API requests and WEBrick as the HTTP server.

## Installation & Usage
Ensure the server running this script has access to Proxmox API.
### Creating a Proxmox User:
1. Navigate to **Datacenter → Permissions → Users** in the Proxmox UI.
2. Click **Add** and create a new user:
   - **User**: `username`
   - **Realm**: `PVE`
   - **Password**: Set a secure password.

### Assigning Permissions:
The user needs specific privileges to read VM and infrastructure data. Assign the following roles to the user:

- `VM.Audit`

- `VM.Config.Disk`

- `Datastore.AllocateSpace`


## Triggering Webhook 

In order to connect this with your Alertmanager it is mandatory to create this labels. 
You can check my automatic PVE scraper [pve-scraper](https://github.com/vvvesss/pve-scraper)
We can simulate Alertmanager webhook desired fields with this simple curl command:

```
curl -X POST http://pvm-disk-resizer.pvm-disk-resizer.svc.cluster.local:8000 -H "Content-Type: application/json" -d '{
  "commonLabels": {
    "mountpoint": "/media/data",
    "device": "/dev/sdb1",
    "ip": "192.168.111.100",
    "pvehost": "proxmox-node",
    "instance": "test-vm",
    "project": "infra",
    "vm_id": "100"
  }
}'
```

This fields are mandatory or you can rewrite my Ruby code to suits your case.

## Alertmanager configuration


### Triggering rule example
```
groups:
- name: compute-engine
  rules:
  - alert: HostOutOfDiskSpace
    expr: (node_filesystem_avail_bytes * 100) / node_filesystem_size_bytes < 10 and node_filesystem_avail_bytes < 214748364800 and ON (instance, device, mountpoint) node_filesystem_readonly == 0
    for: 10m
    labels:
      severity: warning
      type: compute-engine
    annotations:
      summary: Host low disk space {{ $labels.project }} {{ $labels.instance }}
      description: "Disk full < 10% left VALUE = {{ $value | printf \"%.2f\" }}% \n"
```

#### Alert Details:

This Prometheus alert rule monitors disk usage on Compute Engine instances and triggers a warning alert if:

- Available disk space is below 10%, 
- It is less than 200 GiB (214748364800 bytes) of free space remains
- The filesystem is not in read-only mode.
- Trigger Condition: Disk usage exceeds 90% for 10 minutes.
- Severity: warning
- Labels: Includes project and instance details.
- Notification: Summarizes the affected instance and available space.

### Alertmanager receiver webhook config example 
```
route:
  routes:
  - receiver: pve-resizer-web-hook
    matchers:
    - alertname="HostOutOfDiskSpace"
    - device!~"/dev/root"
    - job="pvm"
    continue: true
receivers:
- name: pve-resizer-web-hook
  webhook_configs:
  - send_resolved: false
    http_config:
      follow_redirects: true
    url: http://pvm-disk-resizer.pvm-disk-resizer.svc.cluster.local:8000
    max_alerts: 1

```
Note that this url is only reachable if your Prometheus isntance is in the same kubernetes cluster. If it isn't you should setup ingress for pvm-disk-resizer service.


## Hashicorp Vault credentials setup
Here are **HashiCorp Vault CLI** commands to create the required secrets:

### **1. Enable and Configure KV Secrets Engine**
Ensure the KV secrets engine is enabled:
```sh
vault secrets enable -path=secret kv-v2
vault secrets enable -path=ssh-key kv
```

---

### **2. Store Credentials for `pvm-disk-resizer`**
```sh
vault kv put secret/pvm-disk-resizer/credentials \
    username="your-username" \
    password="your-password" \
    slack="your-slack-webhook"
```

---

### **3. Store SSH Private Key**
```sh
vault kv put ssh-key/pve-vm-private-key key="$(cat /path/to/private-key.pem)"
```

---

### **4. Create Vault Role for `pvm-disk-resizer`**
```sh
vault policy write pvm-disk-resizer - <<EOF
path "secret/pvm-disk-resizer/credentials" {
  capabilities = ["read"]
}
path "ssh-key/pve-vm-private-key" {
  capabilities = ["read"]
}
EOF
```

---

### **5. Assign the Role to a Vault Token**
```sh
vault write auth/kubernetes/role/pvm-disk-resizer \
    bound_service_account_names=pvm-disk-resizer-sa \
    bound_service_account_namespaces=default \
    policies=pvm-disk-resizer \
    ttl=1h
```

---

### **Summary**
- **`vault kv put`** stores secrets for credentials and SSH keys.  
- **Vault policy grants access** to these secrets.  
- **The Kubernetes role (`pvm-disk-resizer`)** allows a service account to read secrets via Vault Agent Injector.  




