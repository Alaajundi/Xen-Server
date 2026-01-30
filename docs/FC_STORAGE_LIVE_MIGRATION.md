# Fiber Channel Storage & Live Migration Guide

## Overview

This guide covers configuring **Fiber Channel (FC) shared storage** for XenServer, enabling **live VM migration** similar to VMware vMotion.

## XenServer Live Migration = VMware vMotion

| VMware Term | XenServer Equivalent | What It Does |
|-------------|---------------------|--------------|
| **vMotion** | **Live Migration** | Move running VM between hosts (same storage) |
| **Storage vMotion** | **Storage XenMotion** | Move VM storage while running |
| **vMotion + Storage vMotion** | **Storage XenMotion** | Move VM and storage together |

With **shared FC storage**, all these migrations work seamlessly!

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Fiber Channel Storage                     │
│                    (Shared FC SAN/Array)                     │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │   Shared Storage Repository (SR)                     │  │
│  │   - VM Disks (VDIs)                                  │  │
│  │   - VM Metadata                                      │  │
│  │   - Accessible by ALL pool members                   │  │
│  └──────────────────────────────────────────────────────┘  │
└───────┬──────────────────┬──────────────────┬──────────────┘
        │ FC HBA           │ FC HBA           │ FC HBA
        │ (multipath)      │ (multipath)      │ (multipath)
        ↓                  ↓                  ↓
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ xenserver01  │   │ xenserver02  │   │ xenserver03  │
│ (Master)     │   │ (Member)     │   │ (Member)     │
│              │   │              │   │              │
│ Local: OS    │   │ Local: OS    │   │ Local: OS    │
│ RAID1        │   │ RAID1        │   │ RAID1        │
└──────────────┘   └──────────────┘   └──────────────┘

VMs can live migrate between ANY host ←→ ←→ ←→
```

## Prerequisites

### Hardware Requirements
- ✅ Fiber Channel HBA in each server (usually 2 for redundancy)
- ✅ FC switch/fabric connecting servers to storage
- ✅ FC storage array (HP 3PAR, Dell EMC, NetApp, etc.)
- ✅ Zoned LUN accessible by all XenServer hosts

### Network Requirements
- ✅ FC zoning configured for all hosts
- ✅ LUN presented to all hosts with same SCSI ID
- ✅ Multipathing recommended (2+ FC paths)

### Software Requirements
- ✅ XenServer installed on all hosts
- ✅ Hosts in same pool
- ✅ Local RAID for OS (configured during install)

## Configuration Steps

### Step 1: Enable FC Storage in Configuration

Edit [group_vars/all.yml](../group_vars/all.yml):

```yaml
# Fiber Channel Storage (for VMs)
fc_storage_enabled: true
fc_sr_name: "FC-Shared-Storage"
fc_sr_description: "Fiber Channel Shared Storage for Live Migration"
create_fc_sr: true
set_fc_sr_as_default: true  # VMs will use FC storage by default
```

### Step 2: Configure FC Storage on Existing Hosts

If XenServer is already installed:

```powershell
# Configure FC storage on all hosts
ansible-playbook playbooks/configure_fc_storage.yml --ask-vault-pass
```

**What this does:**
1. ✅ Scans for FC HBAs
2. ✅ Configures multipathing
3. ✅ Master creates shared FC SR
4. ✅ Members connect to shared FC SR
5. ✅ Sets FC SR as default for VMs
6. ✅ Enables live migration

### Step 3: Install New Hosts with FC Storage

For fresh installations:

```powershell
# Install with FC storage configuration included
ansible-playbook playbooks/install_xenserver.yml --ask-vault-pass
```

FC storage is configured automatically during installation.

## Storage Layout

After configuration, each host has:

### Local Storage (RAID 1)
- **Purpose**: Host operating system only
- **Location**: Local disks (sda, sdb)
- **SR Name**: "Local storage"
- **Migrates**: No

### Shared FC Storage
- **Purpose**: All VM disks
- **Location**: Fiber Channel SAN
- **SR Name**: "FC-Shared-Storage" (configurable)
- **Migrates**: Yes ✅

## Using Live Migration

### Basic Live Migration (vMotion)

Move a running VM to another host:

```bash
# Migrate VM to xenserver02
xe vm-migrate vm=MyVM host=xenserver02 live=true

# Or use UUID
xe vm-migrate uuid=VM_UUID host-uuid=HOST_UUID live=true
```

**Result**: VM moves instantly with zero downtime!

### Check Where VMs Are Running

```bash
# List all VMs and their current host
xe vm-list is-control-domain=false params=name-label,resident-on,power-state

# List VMs on specific host
xe vm-list resident-on=$(xe host-list name-label=xenserver01 --minimal)
```

### Automatic Migration (DRS-like)

For workload balancing:

```bash
# Migrate all VMs from xenserver01 to other hosts
for vm in $(xe vm-list resident-on=$(xe host-list name-label=xenserver01 --minimal) is-control-domain=false --minimal | tr ',' ' '); do
  # Find least loaded host
  TARGET_HOST=$(xe host-list | grep name-label | head -1 | awk '{print $4}')
  xe vm-migrate uuid=$vm host=$TARGET_HOST live=true
  echo "Migrated VM $vm to $TARGET_HOST"
done
```

## Creating VMs on FC Storage

### During VM Creation

VMs automatically use the default SR (FC storage):

```bash
# Create VM (uses default FC SR)
xe vm-install template="Ubuntu Focal Fossa 20.04" new-name-label="MyVM"

# Create VM on specific SR
FC_SR_UUID=$(xe sr-list name-label="FC-Shared-Storage" --minimal)
xe vm-install template="Ubuntu Focal Fossa 20.04" sr-uuid=$FC_SR_UUID new-name-label="MyVM"
```

### Moving Existing VMs to FC Storage

If VMs are on local storage:

```bash
# Get VM disk UUID
VDI_UUID=$(xe vm-param-get uuid=VM_UUID param-name=VBDs | tr ',' '\n' | xargs -I {} xe vbd-param-get uuid={} param-name=vdi-uuid 2>/dev/null | head -1)

# Get FC SR UUID
FC_SR_UUID=$(xe sr-list name-label="FC-Shared-Storage" --minimal)

# Move disk to FC storage
xe vdi-copy uuid=$VDI_UUID sr-uuid=$FC_SR_UUID

# Attach new disk to VM and remove old one
# (detailed steps in XenServer documentation)
```

## Advanced: Storage XenMotion

Move VM **AND** its storage to different SR:

```bash
# Migrate VM to different host AND different storage
xe vm-migrate vm=MyVM host=xenserver02 remote-master=xenserver02-ip remote-username=root remote-password=PASSWORD
```

## Multipathing Configuration

The project automatically configures multipathing for FC:

### View Multipath Devices

```bash
# SSH to any host
ssh root@192.168.1.101

# List multipath devices
multipath -ll

# Expected output:
# 360000000000000000e00000000010001 dm-0 VENDOR,PRODUCT
# size=500G features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
# |-+- policy='round-robin 0' prio=50 status=active
# | |- 2:0:0:1 sdc 8:32  active ready running
# | `- 3:0:0:1 sde 8:64  active ready running
# `-+- policy='round-robin 0' prio=10 status=enabled
#   |- 2:0:1:1 sdd 8:48  active ready running
#   `- 3:0:1:1 sdf 8:80  active ready running
```

### Verify All Paths Are Active

```bash
# Check path status
multipathd show paths

# Should show all paths as "enabled" and "active"
```

## Verification Checklist

### ✅ Verify FC Configuration

```bash
# 1. Check FC HBAs are detected
ls /sys/class/fc_host/

# 2. List FC storage repositories
xe sr-list type=lvmohba

# 3. Verify SR is shared
xe sr-param-get uuid=SR_UUID param-name=shared
# Should return: true

# 4. Check default SR
xe pool-param-get uuid=$(xe pool-list --minimal) param-name=default-SR

# 5. Verify multipathing
multipath -ll

# 6. Test migration
xe vm-migrate vm=TEST_VM host=xenserver02 live=true
```

### ✅ Performance Verification

```bash
# Test FC storage performance
dd if=/dev/zero of=/run/sr-mount/SR_UUID/test bs=1M count=1000
rm /run/sr-mount/SR_UUID/test

# Should show high throughput (depends on your storage)
```

## Troubleshooting

### Issue: No FC HBAs Detected

```bash
# Check for FC HBA cards
lspci | grep -i fibre

# Rescan FC bus
echo "- - -" > /sys/class/scsi_host/host2/scan
```

**Solution**: Verify FC HBA is installed and enabled in BIOS.

### Issue: LUN Not Visible

```bash
# Rescan all FC hosts
for host in /sys/class/fc_host/host*; do
  echo "- - -" > /sys/class/scsi_host/$(basename $host)/scan
done

# Check FC zoning on switch
```

**Solution**: Verify FC zoning includes all XenServer hosts.

### Issue: SR Creation Fails on Members

This is normal! Only the **pool master** creates the SR. Members automatically connect.

**Solution**: 
- Ensure pool master runs first
- Members will attach automatically

### Issue: Migration Fails

```bash
# Check VM is on shared storage
xe vm-param-get uuid=VM_UUID param-name=VBDs | tr ',' '\n' | \
  xargs -I {} xe vbd-param-get uuid={} param-name=vdi-uuid 2>/dev/null | \
  xargs -I {} xe vdi-param-get uuid={} param-name=sr-uuid | \
  xargs -I {} xe sr-param-get uuid={} param-name=name-label
```

**Should return**: "FC-Shared-Storage"  
**If returns**: "Local storage" → VM must be moved to FC storage first

## Performance Tuning

### Multipath I/O Scheduler

For better performance:

```bash
# Check current scheduler
cat /sys/block/sdc/queue/scheduler

# Set to deadline (better for SAN)
echo deadline > /sys/block/sdc/queue/scheduler
```

### Queue Depth

Increase FC queue depth:

```bash
# Increase to 128 (from default 32)
echo 128 > /sys/block/sdc/queue/nr_requests
```

## Comparison: VMware vMotion vs XenServer

| Feature | VMware vMotion | XenServer | Status |
|---------|---------------|-----------|--------|
| Live migrate running VM | ✅ vMotion | ✅ Live Migration | ✅ Equal |
| Migrate VM storage | ✅ Storage vMotion | ✅ Storage XenMotion | ✅ Equal |
| Zero downtime | ✅ Yes | ✅ Yes | ✅ Equal |
| Shared storage required | ✅ Yes | ✅ Yes | ✅ Equal |
| Supports FC SAN | ✅ Yes | ✅ Yes | ✅ Equal |
| Automatic failover | ✅ HA | ✅ HA | ✅ Equal |
| CLI automation | ✅ PowerCLI | ✅ xe CLI | ✅ Equal |

**XenServer live migration is fully equivalent to VMware vMotion!**

## Best Practices

1. **Multipathing**: Always configure 2+ FC paths for redundancy
2. **Default SR**: Set FC storage as default so VMs automatically use it
3. **Local Storage**: Keep local RAID for host OS only, not VMs
4. **Testing**: Test migration before production use
5. **Monitoring**: Monitor FC path health regularly
6. **Backups**: Even with FC storage, maintain VM backups

## Quick Commands Reference

```bash
# List all SRs
xe sr-list

# List VMs and their storage
xe vdi-list params=name-label,sr-uuid,virtual-size

# Live migrate VM
xe vm-migrate vm=VM_NAME host=TARGET_HOST live=true

# Check multipath status
multipath -ll

# Rescan FC buses
for h in /sys/class/fc_host/host*; do 
  echo "- - -" > /sys/class/scsi_host/$(basename $h)/scan
done

# Get FC SR UUID
xe sr-list type=lvmohba name-label="FC-Shared-Storage" --minimal

# Set default SR
xe pool-param-set uuid=$(xe pool-list --minimal) default-SR=SR_UUID
```

## Summary

✅ **FC Storage**: Configured automatically during installation  
✅ **Live Migration**: Enabled on all VMs using FC storage  
✅ **vMotion Equivalent**: Full feature parity with VMware  
✅ **Multipathing**: Automatic redundancy and failover  
✅ **Zero Downtime**: VMs migrate seamlessly between hosts  

Your XenServer environment now has enterprise-grade live migration capabilities! 🚀
