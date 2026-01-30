# Multiple VLAN Configuration Guide

## Overview

Configure **multiple VLANs** on the data network (bond1) for network segmentation. Each VLAN gets its own network with automatic bonding across eth2 and eth3.

## Configuration

### Define Multiple VLANs

In each host's `inventory/host_vars/HOSTNAME.yml`, use the **`data_vlans`** variable:

```yaml
# Data Network VLANs (multiple VLANs on data bond)
data_vlans:
  - vlan_id: 200
    network_name: "VM-Network-VLAN200"
    description: "Production VM Network"
    
  - vlan_id: 201
    network_name: "VM-Network-VLAN201"
    description: "Development VM Network"
    
  - vlan_id: 202
    network_name: "VM-Network-VLAN202"
    description: "DMZ Network"
    
  - vlan_id: 300
    network_name: "VM-Network-Storage"
    description: "Storage Network for VMs"
```

### Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `vlan_id` | VLAN tag (1-4094) | `200` |
| `network_name` | XenServer network name | `"VM-Network-VLAN200"` |
| `description` | Network description | `"Production VMs"` |

### Network Architecture

```
┌─────────────────────────────────────────────────────┐
│  XenServer Host                                     │
│                                                     │
│  ┌──────────────┐              ┌──────────────┐   │
│  │ eth2         │              │ eth3         │   │
│  │ (Physical)   │              │ (Physical)   │   │
│  └──────┬───────┘              └──────┬───────┘   │
│         │                             │            │
│         ├── VLAN 200 ─────────────────┤            │
│         │   (bond1.200)               │            │
│         │   Network: VM-Network-VLAN200            │
│         │                                          │
│         ├── VLAN 201 ─────────────────┤            │
│         │   (bond1.201)               │            │
│         │   Network: VM-Network-VLAN201            │
│         │                                          │
│         ├── VLAN 202 ─────────────────┤            │
│         │   (bond1.202)               │            │
│         │   Network: VM-Network-VLAN202            │
│         │                                          │
│         └── VLAN 300 ─────────────────┘            │
│             (bond1.300)                            │
│             Network: VM-Network-Storage            │
│                                                     │
│  Each VLAN gets LACP bond (802.3ad) for redundancy │
└─────────────────────────────────────────────────────┘
```

## Example Configurations

### Example 1: Basic 3-VLAN Setup

**Use case**: Production, Development, DMZ

```yaml
data_vlans:
  - vlan_id: 100
    network_name: "Production"
    description: "Production VMs"
    
  - vlan_id: 200
    network_name: "Development"
    description: "Development and Testing"
    
  - vlan_id: 300
    network_name: "DMZ"
    description: "Public-facing services"
```

### Example 2: Multi-Tenant Setup

**Use case**: Multiple customers/departments

```yaml
data_vlans:
  - vlan_id: 1010
    network_name: "Tenant-CustomerA"
    description: "Customer A isolated network"
    
  - vlan_id: 1020
    network_name: "Tenant-CustomerB"
    description: "Customer B isolated network"
    
  - vlan_id: 1030
    network_name: "Tenant-CustomerC"
    description: "Customer C isolated network"
    
  - vlan_id: 1100
    network_name: "Shared-Services"
    description: "Shared services for all tenants"
```

### Example 3: Application Tiers

**Use case**: 3-tier application architecture

```yaml
data_vlans:
  - vlan_id: 210
    network_name: "App-Web-Tier"
    description: "Web servers"
    
  - vlan_id: 220
    network_name: "App-Application-Tier"
    description: "Application servers"
    
  - vlan_id: 230
    network_name: "App-Database-Tier"
    description: "Database servers"
    
  - vlan_id: 240
    network_name: "App-Backend"
    description: "Backend services"
```

### Example 4: Service-Based VLANs

**Use case**: Different service types

```yaml
data_vlans:
  - vlan_id: 500
    network_name: "VM-Compute"
    description: "General compute VMs"
    
  - vlan_id: 501
    network_name: "VM-Storage"
    description: "Storage VMs (NFS, iSCSI)"
    
  - vlan_id: 502
    network_name: "VM-Management"
    description: "Management and monitoring VMs"
    
  - vlan_id: 503
    network_name: "VM-Backup"
    description: "Backup infrastructure"
```

## Applying Configuration

### During Initial Installation

VLANs are automatically configured during installation:

```powershell
# Install with multiple VLANs configured
ansible-playbook playbooks/install_xenserver.yml --ask-vault-pass
```

### On Existing Hosts

Apply to already-installed hosts:

```powershell
# Reconfigure network with new VLANs
ansible-playbook playbooks/install_xenserver.yml \
  --limit xenserver01 \
  --tags postconfig,network,vlan \
  --ask-vault-pass
```

### Add VLANs to Running System

1. **Edit host_vars** - Add new VLANs to `data_vlans` array
2. **Run postconfig**:
   ```powershell
   ansible-playbook playbooks/install_xenserver.yml \
     --limit xenserver01 \
     --tags postconfig,vlan \
     --ask-vault-pass
   ```

## Verification

### Check Created Networks

```bash
# SSH to host
ssh root@192.168.1.101

# List all networks
xe network-list params=name-label,name-description,uuid

# Should show all VLAN networks
```

### Verify VLANs

```bash
# List VLANs
xe vlan-list params=tag,untagged-PIF,tagged-PIF

# Check specific VLAN
xe vlan-list tag=200
```

### Check Bonds

```bash
# List all bonds
xe bond-list params=uuid,master,mode

# Verify each VLAN has a bond
xe bond-list | grep -A 10 "VLAN"
```

### Verify Network Connectivity

```bash
# List PIFs and their VLANs
xe pif-list params=device,VLAN,network-name-label

# Check bond status
xe pif-list params=device,currently-attached,bond-master-of
```

## Using VLANs with VMs

### Create VM on Specific VLAN

```bash
# Get network UUID for VLAN 200
NET_UUID=$(xe network-list name-label="VM-Network-VLAN200" --minimal)

# Create VM
VM_UUID=$(xe vm-install template="Ubuntu Focal Fossa 20.04" new-name-label="WebServer01")

# Attach to VLAN 200 network
xe vif-create vm-uuid=$VM_UUID network-uuid=$NET_UUID device=0

# Start VM
xe vm-start uuid=$VM_UUID
```

### Move VM to Different VLAN

```bash
# Get current VIF
VIF_UUID=$(xe vif-list vm-uuid=$VM_UUID --minimal)

# Destroy old VIF
xe vif-destroy uuid=$VIF_UUID

# Create new VIF on different network
NET_UUID=$(xe network-list name-label="VM-Network-VLAN201" --minimal)
xe vif-create vm-uuid=$VM_UUID network-uuid=$NET_UUID device=0
```

### List VMs by VLAN

```bash
# Show all VMs and their networks
xe vm-list params=name-label,networks

# Filter by specific network
xe vif-list network-name-label="VM-Network-VLAN200" params=vm-name-label
```

## Switch Configuration

### Required Switch Settings

For multiple VLANs on trunk ports:

```
interface TenGigabitEthernet1/0/1
  description XenServer-eth2
  switchport mode trunk
  switchport trunk allowed vlan 200,201,202,300
  spanning-tree portfast trunk
  channel-group 1 mode active  # For LACP
```

```
interface TenGigabitEthernet1/0/2
  description XenServer-eth3
  switchport mode trunk
  switchport trunk allowed vlan 200,201,202,300
  spanning-tree portfast trunk
  channel-group 1 mode active  # For LACP
```

### LACP Configuration

```
interface port-channel1
  description XenServer-Bond1
  switchport mode trunk
  switchport trunk allowed vlan 200,201,202,300
```

## Advanced Configuration

### Per-Host Different VLANs

Each host can have different VLANs if needed:

**xenserver01** (Production host):
```yaml
data_vlans:
  - vlan_id: 100
    network_name: "Production-Web"
    description: "Web tier"
  - vlan_id: 101
    network_name: "Production-DB"
    description: "Database tier"
```

**xenserver02** (Development host):
```yaml
data_vlans:
  - vlan_id: 200
    network_name: "Dev-Environment"
    description: "Development VMs"
  - vlan_id: 201
    network_name: "Test-Environment"
    description: "Testing VMs"
```

### Global VLAN Defaults

Set default VLANs in `group_vars/all.yml`:

```yaml
# Default VLANs (can be overridden per-host)
default_data_vlans:
  - vlan_id: 200
    network_name: "VM-Network-VLAN200"
    description: "Production VM Network"
  - vlan_id: 201
    network_name: "VM-Network-VLAN201"
    description: "Development VM Network"
```

Then in host_vars:
```yaml
# Use defaults or override
data_vlans: "{{ default_data_vlans }}"

# Or customize per host
data_vlans:
  - vlan_id: 300
    network_name: "Custom-Network"
    description: "Host-specific network"
```

## Troubleshooting

### VLANs Not Created

**Check:**
```bash
# Verify eth2 and eth3 are up
xe pif-list device=eth2 params=currently-attached
xe pif-list device=eth3 params=currently-attached

# Should show: currently-attached ( RO): true
```

**Fix:**
```bash
# Plug interfaces
xe pif-plug uuid=$(xe pif-list device=eth2 --minimal)
xe pif-plug uuid=$(xe pif-list device=eth3 --minimal)
```

### Bond Not Created

**Check:**
```bash
# List VLAN PIFs for specific VLAN
xe vlan-list tag=200
```

**Fix:**
Ensure both eth2 and eth3 have VLAN PIFs before bonding.

### VM Cannot Reach Network

**Check:**
```bash
# Verify VM is on correct network
VM_UUID="your-vm-uuid"
xe vif-list vm-uuid=$VM_UUID params=network-name-label

# Check if network is attached to correct VLAN
xe network-param-get uuid=$NET_UUID param-name=name-label
```

### Switch Shows VLAN Mismatch

**Verify:** VLAN IDs match on both XenServer and switch configuration.

```bash
# XenServer side
xe vlan-list params=tag

# Should match switch trunk allowed VLANs
```

## Best Practices

1. **Consistent VLANs**: Use same VLAN IDs across all pool members
2. **Naming Convention**: Use descriptive network names (include VLAN ID)
3. **Documentation**: Document which VLANs are for which purpose
4. **Switch Config**: Configure switches before applying to XenServer
5. **Test VLANs**: Test one VLAN before adding many
6. **VLAN Range**: Plan VLAN ID ranges (e.g., 100-199 for prod, 200-299 for dev)

## VLAN Planning Template

```yaml
# VLAN Planning Template
# Copy to your host_vars and customize

data_vlans:
  # Production VLANs (100-199)
  - vlan_id: 100
    network_name: "Prod-Frontend"
    description: "Production frontend services"
    
  - vlan_id: 101
    network_name: "Prod-Backend"
    description: "Production backend services"
    
  - vlan_id: 102
    network_name: "Prod-Database"
    description: "Production databases"
    
  # Development VLANs (200-299)
  - vlan_id: 200
    network_name: "Dev-General"
    description: "General development"
    
  - vlan_id: 201
    network_name: "Dev-Testing"
    description: "QA and testing"
    
  # Management VLANs (300-399)
  - vlan_id: 300
    network_name: "Mgmt-Monitoring"
    description: "Monitoring and logging"
    
  - vlan_id: 301
    network_name: "Mgmt-Backup"
    description: "Backup infrastructure"
    
  # DMZ VLANs (400-499)
  - vlan_id: 400
    network_name: "DMZ-Public"
    description: "Public-facing services"
```

## Migration from Single VLAN

If you previously had single VLAN configuration:

```yaml
# Old config (still works for backward compatibility)
data_vlan: 200
data_network_name: "VM-Network"
```

Migrate to multiple VLANs:

```yaml
# New config
data_vlans:
  - vlan_id: 200  # Keep your existing VLAN
    network_name: "VM-Network-VLAN200"
    description: "Legacy VM Network"
  - vlan_id: 201  # Add new VLANs
    network_name: "VM-Network-VLAN201"
    description: "Additional network"

# Keep old vars for backward compatibility
data_vlan: 200
data_network_name: "VM-Network"
```

---

**You now have flexible multiple VLAN support!** 🎯 Each VLAN gets its own bonded network for maximum redundancy and performance.
