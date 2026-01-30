# SSH Keys and Pool Master Configuration Guide

## 🔑 SSH Key Management

### How SSH Keys Work in This Project

1. **During Installation**: SSH keys are deployed via the post-installation script
2. **After Installation**: Ansible connects using the deployed key (if configured)
3. **Fallback**: Password authentication remains available unless explicitly disabled

### Setting Up SSH Keys

#### Step 1: Generate SSH Key (if you don't have one)

```powershell
# On Windows (PowerShell)
ssh-keygen -t rsa -b 4096 -C "ansible@xenserver"

# On Linux/Mac
ssh-keygen -t rsa -b 4096 -C "ansible@xenserver"
```

This creates:
- Private key: `~/.ssh/id_rsa` (keep secure!)
- Public key: `~/.ssh/id_rsa.pub` (deploy to servers)

#### Step 2: Add Public Key to Vault

Edit your vault file:
```powershell
ansible-vault edit group_vars/vault.yml
```

Add your public key:
```yaml
---
vault_ilo_username: Administrator
vault_ilo_password: YourIloPassword
vault_xenserver_root_password: YourXenServerPassword

# Add this - paste the ENTIRE content of your ~/.ssh/id_rsa.pub file
vault_ssh_public_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC... your_key ansible@xenserver"
```

#### Step 3: Configure SSH Key Deployment

In [group_vars/all.yml](../group_vars/all.yml):

```yaml
# SSH Key Configuration
ssh_key_deployment: true  # Enable SSH key deployment
ssh_public_key: "{{ vault_ssh_public_key }}"
ssh_disable_password_auth: false  # Set true for key-only (more secure)
```

#### Step 4: Configure Ansible to Use SSH Keys

Update [group_vars/xenservers.yml](../group_vars/xenservers.yml):

```yaml
# Post-installation SSH settings
ansible_port: 22
ansible_user: root
ansible_ssh_private_key_file: ~/.ssh/id_rsa  # Path to your private key
```

Or in [ansible.cfg](../ansible.cfg):
```ini
[defaults]
private_key_file = ~/.ssh/id_rsa
```

### SSH Authentication Options

#### Option 1: Password + SSH Key (Default - Recommended for initial setup)
```yaml
ssh_key_deployment: true
ssh_disable_password_auth: false
```
- ✅ Most flexible
- ✅ Fallback to password if key fails
- ⚠️ Less secure (password still enabled)

#### Option 2: SSH Key Only (Most Secure)
```yaml
ssh_key_deployment: true
ssh_disable_password_auth: true
```
- ✅ Most secure
- ✅ No password authentication
- ⚠️ Lose SSH key = locked out

#### Option 3: Password Only (Not Recommended)
```yaml
ssh_key_deployment: false
```
- ⚠️ Less secure
- ⚠️ Password in playbooks
- ✅ Simple for testing

### Testing SSH Key Authentication

After installation:

```powershell
# Test SSH connection with key
ssh -i ~/.ssh/id_rsa root@192.168.1.101

# Test Ansible connection
ansible xenservers -m ping --private-key ~/.ssh/id_rsa
```

---

## 🏊 Pool Master Configuration

### Understanding XenServer Pools

A **XenServer Pool** is a cluster of XenServer hosts:
- **Pool Master**: Central management server (required)
- **Pool Members**: Additional servers that join the master
- All managed as a single unit
- Shared storage and networking
- Live migration between hosts

### Setting the Pool Master

#### In Host Variables

The pool master is configured **per-host** in `inventory/host_vars/`:

**Pool Master** (`inventory/host_vars/xenserver01.yml`):
```yaml
# XenServer Specific
xenserver_hostname: xenserver01.example.com

# Pool Configuration
xenserver_pool_master: true        # ← This makes it the master
xenserver_pool_role: "master"
```

**Pool Members** (`inventory/host_vars/xenserver02.yml`, `xenserver03.yml`):
```yaml
# XenServer Specific
xenserver_hostname: xenserver02.example.com

# Pool Configuration
xenserver_pool_master: false       # ← This makes it a member
xenserver_pool_role: "member"
pool_master_ip: 192.168.1.101     # ← IP of the master server
```

#### In Global Variables

Pool settings in [group_vars/all.yml](../group_vars/all.yml):

```yaml
# XenServer Pool Configuration
create_pool: true                          # Enable pool creation
pool_name: "Production-Pool"               # Pool name in XenCenter
pool_description: "XenServer Production Pool"
```

### Pool Configuration Workflow

#### Automatic (During Installation)

The main installation playbook automatically handles pool configuration:

```powershell
# Installs ALL servers and configures pool
ansible-playbook playbooks/install_xenserver.yml --ask-vault-pass
```

**What happens:**
1. ✅ xenserver01 installs → Creates pool master
2. ✅ xenserver02 installs → Joins pool automatically
3. ✅ xenserver03 installs → Joins pool automatically

#### Manual (After Installation)

If you need to configure pool later:

```powershell
# Configure pool on already-installed servers
ansible-playbook playbooks/configure_pool.yml --ask-vault-pass
```

### Changing the Pool Master

**Scenario**: You want xenserver02 to be the master instead

1. **Edit host_vars files**:

`inventory/host_vars/xenserver01.yml`:
```yaml
xenserver_pool_master: false
xenserver_pool_role: "member"
pool_master_ip: 192.168.1.102  # Point to new master
```

`inventory/host_vars/xenserver02.yml`:
```yaml
xenserver_pool_master: true
xenserver_pool_role: "master"
# Remove pool_master_ip line
```

2. **Reinstall or reconfigure**:
```powershell
ansible-playbook playbooks/configure_pool.yml --ask-vault-pass
```

### Pool Configuration Examples

#### Single Server (No Pool)

```yaml
# group_vars/all.yml
create_pool: false

# host_vars/xenserver01.yml
xenserver_pool_master: true  # Standalone
```

#### 3-Server Pool (Default)

```yaml
# xenserver01 = Master
xenserver_pool_master: true

# xenserver02 = Member
xenserver_pool_master: false
pool_master_ip: 192.168.1.101

# xenserver03 = Member
xenserver_pool_master: false
pool_master_ip: 192.168.1.101
```

#### Multiple Pools

**Pool 1** (Production):
```yaml
# xenserver01.yml
xenserver_pool_master: true
pool_name: "Production-Pool"

# xenserver02.yml
pool_master_ip: 192.168.1.101
```

**Pool 2** (Development):
```yaml
# xenserver03.yml
xenserver_pool_master: true
pool_name: "Dev-Pool"

# xenserver04.yml (if you add it)
pool_master_ip: 192.168.1.103
```

### Verifying Pool Configuration

#### Check Pool Status

```bash
# On the pool master
xe pool-list params=all
xe host-list

# From Ansible
ansible-playbook playbooks/validate_installation.yml --ask-vault-pass
```

#### Expected Output

```
uuid ( RO)                : 12345678-1234-1234-1234-123456789012
name-label ( RW): Production-Pool
name-description ( RW): XenServer Production Pool
master ( RO): 87654321-4321-4321-4321-210987654321

Host List:
uuid: 87654321-4321-4321-4321-210987654321
  name-label: xenserver01.example.com
  address: 192.168.1.101
  enabled: true

uuid: 11111111-2222-3333-4444-555555555555
  name-label: xenserver02.example.com
  address: 192.168.1.102
  enabled: true
```

---

## 🔧 Complete Configuration Example

### Full Setup with SSH Keys and Pool

#### 1. Create Vault with SSH Key

```powershell
ansible-vault create group_vars/vault.yml
```

```yaml
---
# iLO Credentials
vault_ilo_username: Administrator
vault_ilo_password: YourSecureIloPass123!

# XenServer Root Password
vault_xenserver_root_password: SecureXenPass456!

# SSH Public Key (paste entire id_rsa.pub content)
vault_ssh_public_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDGh... ansible@control"
```

#### 2. Configure Global Settings

`group_vars/all.yml`:
```yaml
# SSH Key Configuration
ssh_key_deployment: true
ssh_public_key: "{{ vault_ssh_public_key }}"
ssh_disable_password_auth: false  # false = password fallback enabled

# Pool Configuration
create_pool: true
pool_name: "Production-Pool"
pool_description: "Production XenServer Pool"
```

#### 3. Configure Hosts

`inventory/host_vars/xenserver01.yml`:
```yaml
ilo_ip: 192.168.1.201
management_ip: 192.168.1.101
xenserver_hostname: xenserver01.example.com
xenserver_pool_master: true
xenserver_pool_role: "master"
```

`inventory/host_vars/xenserver02.yml`:
```yaml
ilo_ip: 192.168.1.202
management_ip: 192.168.1.102
xenserver_hostname: xenserver02.example.com
xenserver_pool_master: false
xenserver_pool_role: "member"
pool_master_ip: 192.168.1.101
```

#### 4. Run Installation

```powershell
# Install with SSH keys and pool configuration
ansible-playbook playbooks/install_xenserver.yml --ask-vault-pass

# Test SSH key access after installation
ssh -i ~/.ssh/id_rsa root@192.168.1.101
```

---

## 🎯 Quick Reference

### Where to Configure Each Setting

| Setting | File Location | Purpose |
|---------|--------------|---------|
| **Enable SSH keys** | `group_vars/all.yml` | `ssh_key_deployment: true` |
| **Public key** | `group_vars/vault.yml` | `vault_ssh_public_key: "..."` |
| **Disable password auth** | `group_vars/all.yml` | `ssh_disable_password_auth: true` |
| **Enable pool** | `group_vars/all.yml` | `create_pool: true` |
| **Pool name** | `group_vars/all.yml` | `pool_name: "Production-Pool"` |
| **Set master** | `host_vars/HOST.yml` | `xenserver_pool_master: true` |
| **Set member** | `host_vars/HOST.yml` | `xenserver_pool_master: false` |
| **Master IP** | `host_vars/MEMBER.yml` | `pool_master_ip: 192.168.1.101` |

### Common Commands

```powershell
# Create vault with SSH key
ansible-vault create group_vars/vault.yml

# Edit vault to add SSH key
ansible-vault edit group_vars/vault.yml

# Install with pool and SSH keys
ansible-playbook playbooks/install_xenserver.yml --ask-vault-pass

# Configure pool only (post-install)
ansible-playbook playbooks/configure_pool.yml --ask-vault-pass

# Test SSH key authentication
ssh -i ~/.ssh/id_rsa root@XENSERVER_IP

# Test Ansible with SSH key
ansible xenservers -m ping --private-key ~/.ssh/id_rsa
```

---

## 🐛 Troubleshooting

### SSH Key Issues

**Problem**: Can't connect with SSH key
```powershell
# Check key was deployed
ssh root@192.168.1.101 "cat /root/.ssh/authorized_keys"

# Check key permissions
ssh root@192.168.1.101 "ls -la /root/.ssh/"

# Try verbose SSH
ssh -vvv -i ~/.ssh/id_rsa root@192.168.1.101
```

**Problem**: "Permission denied (publickey)"
- Ensure `ssh_disable_password_auth: false` in `group_vars/all.yml`
- Verify public key is correctly formatted (no line breaks)
- Check private key permissions: `chmod 600 ~/.ssh/id_rsa`

### Pool Issues

**Problem**: Server won't join pool
```powershell
# Check network connectivity to master
ansible xenserver02 -m shell -a "ping -c 4 192.168.1.101"

# Verify master is accessible
ansible xenserver02 -m shell -a "curl http://192.168.1.101"

# Check credentials
ansible xenserver02 -m shell -a "xe host-list server=192.168.1.101 username=root password=PASSWORD"
```

**Problem**: Wrong server is master
```bash
# On any pool member, check current master
xe pool-list params=master --minimal | xargs -I {} xe host-list uuid={} params=address --minimal

# Manually designate new master (emergency)
xe pool-designate-new-master host-uuid=NEW_MASTER_UUID
```

---

This configuration gives you complete control over SSH authentication and pool topology! 🚀
