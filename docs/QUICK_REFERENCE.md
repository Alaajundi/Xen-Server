# Quick Reference: SSH Keys & Pool Master

## 🔑 SSH Keys - Where to Configure

```
┌─────────────────────────────────────────────────────────────┐
│  SSH Key Configuration Flow                                 │
└─────────────────────────────────────────────────────────────┘

1️⃣  Generate Key Pair
    ┌────────────────┐
    │ Control Node   │
    │ ssh-keygen     │
    └────────┬───────┘
             │ Creates:
             │ • ~/.ssh/id_rsa (private - KEEP SECURE!)
             │ • ~/.ssh/id_rsa.pub (public)
             ↓

2️⃣  Store Public Key in Vault
    ┌────────────────────────────────────┐
    │ group_vars/vault.yml (ENCRYPTED)   │
    ├────────────────────────────────────┤
    │ vault_ssh_public_key:              │
    │   "ssh-rsa AAAAB3... your_key"     │
    └────────────────────────────────────┘

3️⃣  Enable SSH Deployment
    ┌────────────────────────────────────┐
    │ group_vars/all.yml                 │
    ├────────────────────────────────────┤
    │ ssh_key_deployment: true           │
    │ ssh_public_key:                    │
    │   "{{ vault_ssh_public_key }}"     │
    │ ssh_disable_password_auth: false   │
    └────────────────────────────────────┘

4️⃣  Configure Ansible to Use Private Key
    ┌────────────────────────────────────┐
    │ group_vars/xenservers.yml          │
    ├────────────────────────────────────┤
    │ ansible_ssh_private_key_file:      │
    │   ~/.ssh/id_rsa                    │
    └────────────────────────────────────┘

5️⃣  Automatic Deployment During Install
    • Ansible generates answer file with SSH key
    • XenServer installer runs post-install script
    • Script creates /root/.ssh/authorized_keys
    • Ready for passwordless SSH! ✅
```

## 🏊 Pool Master - Where to Configure

```
┌─────────────────────────────────────────────────────────────┐
│  Pool Configuration Layout                                  │
└─────────────────────────────────────────────────────────────┘

Global Settings (all pools)
┌────────────────────────────────────┐
│ group_vars/all.yml                 │
├────────────────────────────────────┤
│ create_pool: true                  │
│ pool_name: "Production-Pool"       │
│ pool_description: "Prod Pool"      │
└────────────────────────────────────┘
        │
        ├─────────────────┬─────────────────┬─────────────────┐
        ↓                 ↓                 ↓                 ↓

┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  xenserver01     │  │  xenserver02     │  │  xenserver03     │
│  (MASTER)        │  │  (MEMBER)        │  │  (MEMBER)        │
├──────────────────┤  ├──────────────────┤  ├──────────────────┤
│ host_vars/       │  │ host_vars/       │  │ host_vars/       │
│ xenserver01.yml  │  │ xenserver02.yml  │  │ xenserver03.yml  │
│                  │  │                  │  │                  │
│ ilo_ip:          │  │ ilo_ip:          │  │ ilo_ip:          │
│   192.168.1.201  │  │   192.168.1.202  │  │   192.168.1.203  │
│                  │  │                  │  │                  │
│ management_ip:   │  │ management_ip:   │  │ management_ip:   │
│   192.168.1.101  │  │   192.168.1.102  │  │   192.168.1.103  │
│                  │  │                  │  │                  │
│ xenserver_pool_  │  │ xenserver_pool_  │  │ xenserver_pool_  │
│   master: TRUE ✅│  │   master: false  │  │   master: false  │
│                  │  │                  │  │                  │
│ xenserver_pool_  │  │ xenserver_pool_  │  │ xenserver_pool_  │
│   role: "master" │  │   role: "member" │  │   role: "member" │
│                  │  │                  │  │                  │
│ (no master_ip)   │  │ pool_master_ip:  │  │ pool_master_ip:  │
│                  │  │   192.168.1.101 ─┼──┼──192.168.1.101  │
└──────────────────┘  └──────────────────┘  └──────────────────┘
         │                      │                      │
         │                      ↓                      ↓
         │              Joins pool master    Joins pool master
         │                      │                      │
         └──────────────────────┴──────────────────────┘
                                │
                                ↓
                    ┌────────────────────────┐
                    │   Production-Pool      │
                    │                        │
                    │   Master: xenserver01  │
                    │   Members: 3 hosts     │
                    └────────────────────────┘
```

## 📋 Configuration Checklist

### Before Installation

```
□ Generated SSH key pair (ssh-keygen)
□ Created vault.yml with credentials
  □ vault_ilo_username
  □ vault_ilo_password
  □ vault_xenserver_root_password
  □ vault_ssh_public_key (if using SSH keys)

□ Updated group_vars/all.yml
  □ http_server_ip (your control node IP)
  □ ssh_key_deployment: true (if using SSH keys)
  □ create_pool: true
  □ pool_name

□ Updated each host_vars/HOSTNAME.yml
  □ ilo_ip and ilo_hostname
  □ management_ip and network settings
  □ MAC addresses for all 4 NICs
  □ xenserver_pool_master: true (one server only)
  □ xenserver_pool_master: false (all other servers)
  □ pool_master_ip (for member servers)
  □ RAID disk paths

□ Updated inventory/hosts.yml
  □ Added/removed servers as needed
  □ Set ansible_host for each server
```

### Installation Commands

```powershell
# 1. Setup HTTP server
ansible-playbook playbooks/setup_http_server.yml

# 2. Copy ISO
cp /path/to/XenServer.iso ./http_root/XenServer-8.iso

# 3. Install everything (SSH keys + pool + servers)
ansible-playbook playbooks/install_xenserver.yml --ask-vault-pass

# OR: Install specific server
ansible-playbook playbooks/install_xenserver.yml --limit xenserver01 --ask-vault-pass

# 4. Validate
ansible-playbook playbooks/validate_installation.yml --ask-vault-pass
```

### Post-Installation Testing

```powershell
# Test SSH key authentication
ssh -i ~/.ssh/id_rsa root@192.168.1.101

# Test Ansible connection
ansible xenservers -m ping

# Check pool status (from any server)
ssh root@192.168.1.101 "xe pool-list params=all"
ssh root@192.168.1.101 "xe host-list"
```

## 🎯 Common Scenarios

### Scenario 1: Simple 3-Server Pool with SSH Keys

```yaml
# group_vars/vault.yml
vault_ssh_public_key: "ssh-rsa AAAAB3..."
vault_xenserver_root_password: "SecurePass"

# group_vars/all.yml
ssh_key_deployment: true
create_pool: true
pool_name: "Production-Pool"

# host_vars/xenserver01.yml
xenserver_pool_master: true

# host_vars/xenserver02.yml
xenserver_pool_master: false
pool_master_ip: 192.168.1.101

# host_vars/xenserver03.yml
xenserver_pool_master: false
pool_master_ip: 192.168.1.101
```

**Result**: All servers installed with SSH keys, automatically form pool.

### Scenario 2: Standalone Servers (No Pool)

```yaml
# group_vars/all.yml
create_pool: false  # ← Disable pool

# host_vars/xenserver01.yml
xenserver_pool_master: true  # Standalone

# host_vars/xenserver02.yml
xenserver_pool_master: true  # Standalone

# host_vars/xenserver03.yml
xenserver_pool_master: true  # Standalone
```

**Result**: Three independent XenServer hosts, no pool.

### Scenario 3: Multiple Pools

```yaml
# Pool 1: Production (xenserver01 + xenserver02)
# host_vars/xenserver01.yml
xenserver_pool_master: true
pool_name: "Production-Pool"

# host_vars/xenserver02.yml
xenserver_pool_master: false
pool_master_ip: 192.168.1.101

# Pool 2: Development (xenserver03 + xenserver04)
# host_vars/xenserver03.yml
xenserver_pool_master: true
pool_name: "Dev-Pool"

# host_vars/xenserver04.yml
xenserver_pool_master: false
pool_master_ip: 192.168.1.103
```

**Result**: Two separate pools in same Ansible project.

### Scenario 4: Change Pool Master

**Before**:
- xenserver01 = Master
- xenserver02, xenserver03 = Members

**After** (want xenserver02 as master):

1. **Edit host_vars/xenserver01.yml**:
   ```yaml
   xenserver_pool_master: false
   pool_master_ip: 192.168.1.102  # New master
   ```

2. **Edit host_vars/xenserver02.yml**:
   ```yaml
   xenserver_pool_master: true
   # Remove pool_master_ip line
   ```

3. **Edit host_vars/xenserver03.yml**:
   ```yaml
   pool_master_ip: 192.168.1.102  # New master
   ```

4. **Reconfigure**:
   ```powershell
   ansible-playbook playbooks/configure_pool.yml --ask-vault-pass
   ```

## 🔐 Security Best Practices

### SSH Keys
- ✅ **DO**: Use 4096-bit RSA keys minimum
- ✅ **DO**: Store private key securely (chmod 600)
- ✅ **DO**: Use different keys for different environments
- ✅ **DO**: Keep vault.yml encrypted at all times
- ❌ **DON'T**: Share private keys
- ❌ **DON'T**: Commit private keys to git

### Pool Configuration
- ✅ **DO**: Use strong root passwords (20+ chars)
- ✅ **DO**: Keep vault.yml encrypted
- ✅ **DO**: Use separate pools for different security zones
- ✅ **DO**: Document which server is pool master
- ❌ **DON'T**: Use same password across environments
- ❌ **DON'T**: Store passwords unencrypted

## 📞 Getting Help

**Issue**: SSH key not working
→ See [docs/SSH_KEYS_AND_POOL_MASTER.md](SSH_KEYS_AND_POOL_MASTER.md#ssh-key-issues)

**Issue**: Pool join fails
→ See [docs/SSH_KEYS_AND_POOL_MASTER.md](SSH_KEYS_AND_POOL_MASTER.md#pool-issues)

**Issue**: Wrong server is master
→ See [docs/SSH_KEYS_AND_POOL_MASTER.md](SSH_KEYS_AND_POOL_MASTER.md#changing-the-pool-master)

**General troubleshooting**
→ See [README.md](../README.md#troubleshooting)

---

**Quick tip**: Run `ansible-inventory --graph` to visualize your server groups!
