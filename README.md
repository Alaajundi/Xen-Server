# XenServer Automated Installation for HP DL385 G11

Automated, scalable, unattended XenServer installation on HP ProLiant DL385 Gen11 servers using Ansible. This project uses HTTP-based ISO and answer file delivery, HP iLO for remote management, and RAID 1 storage configuration.

## 🎯 Features

- ✅ **Fully Unattended Installation** - Zero manual intervention required
- ✅ **Per-Host Configuration** - Each server has its own variables in `host_vars/`
- ✅ **RAID 1 Configuration** - Automated RAID 1 setup using all disk space
- ✅ **4-NIC Network Setup** - 2 NICs for management bond, 2 for data bond
- ✅ **HTTP Delivery** - ISO and answer files served over HTTP (no PXE required)
- ✅ **Role-Based Architecture** - Modular, maintainable, and scalable design
- ✅ **HP iLO Integration** - Complete remote control via Redfish API
- ✅ **Serial Installation** - Install one server at a time for easier monitoring
- ✅ **SSH Key Management** - Passwordless authentication with vault-encrypted keys
- ✅ **Pool Configuration** - Automatic pool creation and member joining
- ✅ **Fiber Channel Storage** - Shared FC SAN with live migration (vMotion equivalent)

## 📋 Prerequisites

### Control Node Requirements

- **Ansible**: Version 2.14 or later
- **Python**: 3.8 or later
- **Python Packages**:
  ```bash
  pip install python-hpilo requests
  ```
- **Ansible Collections**:
  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```

### Target Server Requirements

- **Hardware**: HP ProLiant DL385 Gen11
- **iLO**: iLO 6 with Advanced license (for virtual media)
- **Disks**: 2+ disks for RAID 1 configuration
- **Network**: 4 NICs available
- **BIOS**: Configured to allow network boot or CD boot

### Network Requirements

- **Management Network**: VLAN 100 (configurable)
- **Data Network**: VLAN 200 (configurable)
- **HTTP Server**: Accessible from target servers' iLO
- **DNS**: Configured for hostname resolution
- **iLO Network**: Management access to all iLO interfaces

## 🏗️ Project Structure

```
XenServer/
├── ansible.cfg                      # Ansible configuration
├── requirements.yml                 # Required collections
├── .gitignore                       # Git ignore rules
│
├── inventory/
│   ├── hosts.yml                    # Inventory file
│   └── host_vars/                   # Per-host variables
│       ├── xenserver01.yml          # Server 1 config
│       ├── xenserver02.yml          # Server 2 config
│       └── xenserver03.yml          # Server 3 config
│
├── group_vars/
│   ├── all.yml                      # Global variables
│   └── xenservers.yml               # XenServer group variables
│
├── roles/
│   ├── http_server/                 # HTTP server setup
│   │   ├── defaults/main.yml
│   │   └── tasks/main.yml
│   │
│   ├── ilo_management/              # HP iLO control
│   │   ├── defaults/main.yml
│   │   └── tasks/main.yml
│   │
│   ├── xenserver_answerfile/        # Answer file generation
│   │   ├── defaults/main.yml
│   │   ├── tasks/main.yml
│   │   └── templates/
│   │       ├── answerfile.xml.j2
│   │       └── post_install.sh.j2
│   │
│   ├── xenserver_install/           # Installation orchestration
│   │   ├── defaults/main.yml
│   │   └── tasks/main.yml
│   │
│   ├── xenserver_postconfig/        # Post-install configuration
│   │   ├── defaults/main.yml
│   │   └── tasks/main.yml
│   │
│   └── fc_storage/                  # Fiber Channel storage
│       ├── defaults/main.yml
│       ├── tasks/main.yml
│   ├── validate_installation.yml    # Validation playbook
    ├── configure_pool.yml           # Pool configuration
    └── configure_fc_storage.yml     # Fiber Channel storage setup
│       └── handlers/main.yml
│
└── playbooks/
    ├── setup_http_server.yml        # Setup HTTP server
    ├── install_xenserver.yml        # Main installation playbook
    └── validate_installation.yml    # Validation playbook
```

## 🚀 Quick Start

### Step 1: Configure Your Environment

1. **Clone or create the project structure**
   ```bash
   cd "c:\Users\alaaj\OneDrive\Documents\temp vscode\XenServer"
   ```

2. **Install Ansible collections**
   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

3. **Update inventory file** (`inventory/hosts.yml`)
   - Set your control node IP for the HTTP server
   - Update XenServer host IPs

4. **Configure per-host variables** (`inventory/host_vars/*.yml`)
   - Update iLO IPs and credentials
   - Set management IPs and network configuration
   - Update MAC addresses for NICs
   - Configure RAID disk paths

5. **Create Ansible Vault for sensitive data**
   ```bash
   ansible-vault create group_vars/vault.yml
   ```
   
   Add the following content:
   ```yaml
   ---
   vault_ilo_username: Administrator
   vault_ilo_password: YourIloPassword
   vault_xenserver_root_password: YourXenServerPassword
   ```

6. **Update global variables** (`group_vars/all.yml`)
   - Set HTTP server IP to your control node's IP
   - Configure network VLANs
   - Set timezone and NTP servers

### Step 2: Prepare XenServer ISO

1. **Download XenServer ISO** from Citrix website

2. **Setup HTTP server and place ISO**
   ```bash
   ansible-playbook playbooks/setup_http_server.yml
   ```

3. **Copy ISO to HTTP root directory**
   ```bash
   # The setup playbook will tell you where to copy the ISO
   # Default location: ./http_root/XenServer-8.iso
   cp /path/to/your/xenserver.iso ./http_root/XenServer-8.iso
   ```

4. **Verify ISO is accessible**
   ```bash
   curl -I http://YOUR_IP:8080/XenServer-8.iso
   ```

### Step 3: Run Installation

1. **Install XenServer on all servers**
   ```bash
   ansible-playbook playbooks/install_xenserver.yml --ask-vault-pass
   ```
   
   This will:
   - Install XenServer on all servers (one at a time)
   - Deploy SSH keys for passwordless authentication
   - Create pool master on xenserver01
   - Join xenserver02 and xenserver03 to the pool automatically

2. **Install on specific server**
   ```bash
   ansible-playbook playbooks/install_xenserver.yml --limit xenserver01 --ask-vault-pass
   ```

3. **Monitor installation progress**
   - Watch Ansible output for progress
   - Access iLO console: `https://ILO_IP` for visual monitoring
   - Installation typically takes 20-40 minutes per server

### Step 4: Validate Installation

```bash
ansible-playbook playbooks/validate_installation.yml --ask-vault-pass
```

## 📝 Configuration Guide

### SSH Key Authentication

The project supports SSH key-based authentication for secure, passwordless access:

1. **Generate SSH key** (if needed):
   ```powershell
   ssh-keygen -t rsa -b 4096 -C "ansible@xenserver"
   ```

2. **Add public key to vault**:
   ```powershell
   ansible-vault edit group_vars/vault.yml
   ```
   
   Add line:
   ```yaml
   vault_ssh_public_key: "ssh-rsa AAAAB3NzaC... your_key"
   ```

3. **Enable in global config** (`group_vars/all.yml`):
   ```yaml
   ssh_key_deployment: true
   ssh_public_key: "{{ vault_ssh_public_key }}"
   ssh_disable_password_auth: false  # Set true for key-only
   ```

Keys are automatically deployed during installation. See [SSH_KEYS_AND_POOL_MASTER.md](docs/SSH_KEYS_AND_POOL_MASTER.md) for details.

### Pool Master Configuration

Configure which server is the pool master in host_vars:

**Pool Master** (`inventory/host_vars/xenserver01.yml`):
```yaml
xenserver_pool_master: true
xenserver_pool_role: "master"
```

**Pool Members** (`inventory/host_vars/xenserver02.yml`):
```yaml
xenserver_pool_master: false
xenserver_pool_role: "member"
pool_master_ip: 192.168.1.101  # Master's IP
```

PoolFiber Channel Storage (Live Migration)

**Enable shared FC storage** for live VM migration (VMware vMotion equivalent):

```yaml
# In group_vars/all.yml
fc_storage_enabled: true
fc_sr_name: "FC-Shared-Storage"
set_fc_sr_as_default: true  # VMs use FC storage by default
```

**Requirements:**
- Fiber Channel HBA in each server
- FC LUN zoned and accessible by all hosts
- XenServer hosts in same pool

**What you get:**
- ✅ Live VM migration between hosts (like vMotion)
- ✅ Automatic multipathing for redundancy
- ✅ Shared storage for all VMs
- ✅ Local RAID for host OS only

**Configure after installation:**
```bash
ansible-playbook playbooks/configure_fc_storage.yml --ask-vault-pass
```

See [FC_STORAGE_LIVE_MIGRATION.md](docs/FC_STORAGE_LIVE_MIGRATION.md) for complete guide.

###  is automatically created during installation. See [SSH_KEYS_AND_POOL_MASTER.md](docs/SSH_KEYS_AND_POOL_MASTER.md) for advanced pool configuration.

### Per-Host Variables

Each server has its own configuration file in `inventory/host_vars/`. Here's what you need to configure:

```yaml
# iLO Configuration
ilo_hostname: ilo-xenserver01.example.com
ilo_ip: 192.168.1.201
ilo_username: "{{ vault_ilo_username }}"
ilo_password: "{{ vault_ilo_password }}"

# Network Interfaces (UPDATE MAC ADDRESSES!)
network_interfaces:
  - name: eth0
    mac: "AA:BB:CC:DD:EE:01"  # ← UPDATE THIS
    purpose: management
    bond: bond0
    bond_mode: active-backup
  # ... configure all 4 NICs

# Management Network
management_ip: 192.168.1.101
management_netmask: 255.255.255.0
management_gateway: 192.168.1.1

# RAID Configuration
raid_config:
  type: raid1
  disks:
    - /dev/sda
    - /dev/sdb
  use_all_space: true
```

### Network Configuration
Pool Management

#### Reconfigure Pool After Installation

```bash
# Configure or reconfigure pool membership
ansible-playbook playbooks/configure_pool.yml --ask-vault-pass
```

#### Change Pool Master

1. Edit host_vars to change which server is master
2. Run pool configuration:
   ```bash
   ansible-playbook playbooks/configure_pool.yml --ask-vault-pass
   ```

See [docs/SSH_KEYS_AND_POOL_MASTER.md](docs/SSH_KEYS_AND_POOL_MASTER.md) for detailed pool management.

### 
The project configures a 4-NIC setup:

- **Management Bond (bond0)**: 
  - eth0 + eth1
  - Mode: active-backup
  - VLAN: 100 (default)
  
- **Data Bond (bond1)**:
  - eth2 + eth3
  - Mode: 802.3ad (LACP)
  - VLAN: 200 (default)

### Storage Configuration

- **RAID Type**: RAID 1 (mirrored)
- **Disk Usage**: All available space
- **Disks**: Configured in `raid_config.disks` per host

## 🔧 Advanced Usage

### Installing Additional Servers

1. **Add new host configuration**
   ```bash
   cp inventory/host_vars/xenserver01.yml inventory/host_vars/xenserver04.yml
   ```

2. **Update the new host file** with unique values:
   - iLO IP and hostname
   - Management IP
   - MAC addresses
   - Hostname

3. **Add to inventory** (`inventory/hosts.yml`):
   ```yaml
   xenservers:
     hosts:
       xenserver04:
         ansible_host: 192.168.1.104
   ```

4. **Run installation**
   ```bash
   ansible-playbook playbooks/install_xenserver.yml --limit xenserver04 --ask-vault-pass
   ```

### Customizing Installation

#### Change RAID Configuration

Edit `inventory/host_vars/HOSTNAME.yml`:
```yaml
raid_config:
  type: raid1
  disks:
    - /dev/sda
    - /dev/sdb
    - /dev/sdc  # Add more disks
  use_all_space: true
```

#### Change Network VLANs

Edit `group_vars/all.yml`:
```yaml
management_network:
  vlan: 150  # Change from default 100
  mtu: 1500

data_network:
  vlan: 250  # Change from default 200
  mtu: 9000  # Jumbo frames
```

#### Modify Bond Modes

Edit `inventory/host_vars/HOSTNAME.yml`:
```yaml
network_interfaces:
  - name: eth0
    bond_mode: "802.3ad"  # Change to LACP instead of active-backup
```

### Troubleshooting

#### ISO Not Accessible
```bash
# Check HTTP server status
./http_root/check_server.sh

# Restart HTTP server
kill $(cat http_root/http_server.pid)
ansible-playbook playbooks/setup_http_server.yml
```

#### iLO Connection Issues
```bash
# Test iLO connectivity
curl -k https://ILO_IP

# Verify credentials
ansible xenservers -m community.general.redfish_info \
  -a "baseuri={{ ilo_ip }} username={{ ilo_username }} password={{ ilo_password }} category=Systems" \
  --limit xenserver01
```

#### Installation Hangs
- Check iLO console for error messages
- Verify answer file is accessible: `curl http://HTTP_IP:8080/answerfiles/answerfile_HOSTNAME.xml`
- Check RAID configuration is correct for your hardware
- Ensure sufficient disk space

#### Network Configuration Issues
- Verify MAC addresses match physical NICs
- Check switch configuration for LACP/bonding support
- Confirm VLAN configuration on switches
- Verify IP addresses don't conflict

### Tags for Selective Execution
**[SSH Keys & Pool Master Configuration](docs/SSH_KEYS_AND_POOL_MASTER.md)** - Detailed guide
- 
Run specific parts of the installation:

```bash
# Only generate answer files
ansible-playbook playbooks/install_xenserver.yml --tags answerfile

# Only configure iLO and boot
ansible-playbook playbooks/install_xenserver.yml --tags ilo,boot

# Only run post-configuration
ansible-playbook playbooks/install_xenserver.yml --tags postconfig

# Skip network configuration
ansible-playbook playbooks/install_xenserver.yml --skip-tags network
```

## 🔒 Security Best Practices

1. **Use Ansible Vault** for all sensitive data:
   ```bash
   ansible-vault encrypt inventory/host_vars/xenserver01.yml
   ```

2. **Restrict file permissions**:
   ```bash
   chmod 600 inventory/host_vars/*.yml
   chmod 600 group_vars/vault.yml
   ```

3. **Change default passwords** immediately after installation

4. **Secure HTTP server**:
   - Use HTTPS if possible
   - Restrict access to installation network only
   - Stop HTTP server when not in use

5. **iLO Security**:
   - Use strong iLO passwords
   - Enable two-factor authentication
   - Restrict iLO network access

## 📊 Installation Timeline

| Phase | Duration | Description |
|-------|----------|-------------|
| HTTP Setup | 1-2 min | Start HTTP server and prepare files |
| Answer File Generation | <1 min | Generate per-host answer files |
| iLO Configuration | 2-3 min | Mount ISO, configure boot |
| Server Boot | 3-5 min | Server powers on and boots from ISO |
| XenServer Installation | 15-30 min | Automated installation process |
| Post-Install Scripts | 3-5 min | Network and storage configuration |
| Validation | 2-3 min | Verify installation success |
| **Total** | **25-50 min** | Per server (serial installation) |

## 🤝 Contributing

To extend this project:

1. **Add new roles** in `roles/` directory
2. **Create new playbooks** in `playbooks/` directory
3. **Update documentation** in this README
4. **Test thoroughly** before deploying to production

## 📚 Additional Resources

- **[SSH Keys & Pool Master Configuration](docs/SSH_KEYS_AND_POOL_MASTER.md)** - Detailed guide
- **[Fiber Channel Storage & Live Migration](docs/FC_STORAGE_LIVE_MIGRATION.md)** - FC SAN setup
- [XenServer Documentation](https://docs.citrix.com/en-us/citrix-hypervisor/)
- [HP iLO REST API](https://hewlettpackard.github.io/ilo-rest-api-docs/)
- [Ansible Documentation](https://docs.ansible.com/)
- [XenServer Answer File Reference](https://docs.citrix.com/en-us/citrix-hypervisor/install/answerfile.html)

## ⚠️ Important Notes

- **Backup Data**: This installation will **erase all data** on target disks
- **Network Downtime**: Servers will be unavailable during installation
- **Serial Installation**: Servers install one at a time by default (configurable)
- **iLO License**: Advanced iLO license required for virtual media
- **Testing**: Always test in a lab environment first

## 📄 License

This project is provided as-is for educational and production use.

## 🐛 Known Issues

- XML validation requires `xmllint` (optional, install `libxml2-utils` on Linux)
- HTTP server uses Python's built-in server (not suitable for production at scale)
- iLO virtual media may have timeout issues on slow networks

## 📞 Support

For issues and questions:
1. Check the Troubleshooting section above
2. Review Ansible playbook output for error details
3. Check iLO console logs
4. Review generated answer files in `http_root/answerfiles/`

---

**Happy Automating! 🚀**
