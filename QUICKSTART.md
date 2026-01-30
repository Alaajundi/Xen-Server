# XenServer Automated Installation - Quick Reference

## Installation Workflow

```bash
# 1. Setup HTTP Server (one time)
ansible-playbook playbooks/setup_http_server.yml

# 2. Copy XenServer ISO
cp /path/to/XenServer-8.iso ./http_root/XenServer-8.iso

# 3. Install XenServer on all servers
ansible-playbook playbooks/install_xenserver.yml --ask-vault-pass

# 4. Validate installation
ansible-playbook playbooks/validate_installation.yml --ask-vault-pass
```

## Common Commands

```bash
# Install on specific server
ansible-playbook playbooks/install_xenserver.yml --limit xenserver01 --ask-vault-pass

# Check inventory
ansible-inventory --list -i inventory/hosts.yml

# Test connectivity
ansible xenservers -m ping

# Generate answer files only
ansible-playbook playbooks/install_xenserver.yml --tags answerfile

# Run post-config only
ansible-playbook playbooks/install_xenserver.yml --tags postconfig
```

## Files to Customize

1. **inventory/hosts.yml** - Add/remove servers
2. **inventory/host_vars/*.yml** - Per-server configuration
3. **group_vars/all.yml** - Global settings
4. **group_vars/vault.yml** - Encrypted credentials

## Network Setup (4 NICs)

- **eth0 + eth1** → bond0 (Management, active-backup)
- **eth2 + eth3** → bond1 (Data, 802.3ad LACP)

## URLs After Setup

- HTTP Server: http://CONTROL_IP:8080
- ISO: http://CONTROL_IP:8080/XenServer-8.iso
- Answer Files: http://CONTROL_IP:8080/answerfiles/
- iLO Console: https://ILO_IP

## Typical Installation Time

- Single server: 25-50 minutes
- Three servers (serial): 1.5-3 hours
