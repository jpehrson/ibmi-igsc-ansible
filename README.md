# IBM i WebSphere Application Server Deployment Automation with Ansible

Comprehensive Ansible automation framework for deploying IBM WebSphere Application Server v9.0 on IBM i systems using existing shell scripts.

## Version

**Version:** 1.0.0  
**Build Date:** 03/18/2026

## Overview

This Ansible automation framework orchestrates the complete WAS deployment workflow:

1. **Save Operation**: Execute `was_save.sh` on a template system to create WAS profile and HTTP server save files
2. **File Transfer**: Distribute save files from template system to all managed nodes in the target environment
3. **Deployment**: Execute `was_full_deployment.sh` on each managed node with its specific configuration file

### Key Features

- **Template-Based Deployment**: Create save files once on a template system, deploy to multiple nodes
- **Environment-Based Management**: Organize systems by environment (dev, test, prod)
- **1:1 Config Mapping**: Each host has a dedicated configuration file defined in inventory
- **Parallel Execution**: Transfer files and deploy to multiple systems efficiently
- **Comprehensive Logging**: Detailed logs for each operation and host
- **Error Handling**: Robust error detection and reporting
- **IBM i Native**: Uses Qshell (`/usr/bin/sh`) for optimal IBM i compatibility

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Ansible Control Node                         │
│                  (Orchestrates Workflow)                        │
└────────────┬────────────────────────────────────────────────────┘
             │
             ├─── Step 1: Execute was_save.sh ───────────────┐
             │                                                │
             ▼                                                ▼
    ┌────────────────┐                              ┌─────────────┐
    │ Template System│                              │  Save Files │
    │  (ibmi-template)│────────────────────────────▶│  - WASPRODS │
    └────────────────┘                              │  - WEBPRODS1│
                                                    │  - WEBPRODS2│
                                                    └─────────────┘
             │
             ├─── Step 2: Transfer Files ───────────────────┐
             │                                               │
             ▼                                               ▼
    ┌─────────────────────────────────────────────────────────────┐
    │              Managed Nodes (by Environment)                 │
    ├─────────────────┬─────────────────┬─────────────────────────┤
    │   Development   │      Test       │      Production         │
    │  - ibmi-dev1    │  - ibmi-test1   │  - ibmi-prod1          │
    │  - ibmi-dev2    │  - ibmi-test2   │  - ibmi-prod2          │
    └─────────────────┴─────────────────┴─────────────────────────┘
             │
             ├─── Step 3: Deploy WAS ───────────────────────┐
             │                                               │
             ▼                                               ▼
    Each node deploys with its specific config file:
    - ibmi-dev1  → dev1_deployment.config
    - ibmi-dev2  → dev2_deployment.config
    - ibmi-test1 → test1_deployment.config
    - etc.
```

## Directory Structure

```
ansible/
├── ansible.cfg                          # Ansible configuration for IBM i
├── inventory/
│   ├── hosts.yml                        # Inventory with host definitions
│   └── group_vars/
│       ├── all.yml                      # Common variables for all hosts
│       ├── template_systems.yml         # Template system variables
│       ├── dev.yml                      # Development environment variables
│       ├── test.yml                     # Test environment variables
│       └── prod.yml                     # Production environment variables
├── configs/                             # Configuration files directory
│   ├── template_save.config             # Template system save config
│   ├── dev1_deployment.config           # Dev1 deployment config
│   ├── dev2_deployment.config           # Dev2 deployment config
│   ├── test1_deployment.config          # Test1 deployment config
│   ├── test2_deployment.config          # Test2 deployment config
│   ├── prod1_deployment.config          # Prod1 deployment config
│   └── prod2_deployment.config          # Prod2 deployment config
├── playbooks/
│   ├── main.yml                         # Main orchestration playbook
│   ├── was_save.yml                     # Execute save on template system
│   ├── transfer_files.yml               # Transfer save files to managed nodes
│   └── was_deployment.yml               # Deploy WAS on managed nodes
├── logs/                                # Log files (created at runtime)
└── README.md                            # This file
```

## Prerequisites

### Control Node Requirements

- Ansible 2.9 or higher
- Python 3.6 or higher
- SSH access to all IBM i systems
- Network connectivity to all target systems

### IBM i System Requirements

- IBM i 7.4 or higher
- Python 3.9 installed (`/QOpenSys/pkgs/bin/python3.9`) along with python39-itoolkit and python39-ibm_db packages
- Qshell (`/usr/bin/sh`)
- SSH server configured and running
- User profile with appropriate authorities:
  - *ALLOBJ special authority (for save/restore operations)
  - Authority to manage WAS profiles and HTTP servers
  - Authority to create/manage save files

### Required Scripts

The following scripts must be present in the parent directory:
- `../was_save/was_save.sh` - WAS save automation script
- `../was_install_deploy/was_full_deployment.sh` - WAS deployment script

## Installation

1. **Clone or copy the ansible directory** to your control node:
   ```bash
   cd /path/to/WAS_Deployment_Automation
   ```

2. **Configure inventory** - Edit `inventory/hosts.yml`:
   - Update IP addresses for all systems
   - Set appropriate usernames
   - Configure passwords (use ansible-vault for security)
   - Assign config_file for each host

3. **Create configuration files** - Customize files in `configs/`:
   - `template_save.config` - Template system save settings
   - `*_deployment.config` - Deployment settings for each managed node

4. **Encrypt sensitive data** (recommended):
   ```bash
   ansible-vault encrypt_string 'your_password' --name 'vault_dev1_password'
   ```

5. **Test connectivity**:
   ```bash
   ansible all -m ping
   ```

## Configuration

### Inventory Configuration

Each host in the inventory must have:
- `ansible_host`: IP address or hostname
- `ansible_user`: IBM i user profile
- `ansible_ssh_pass`: Password (use vault for security)
- `config_file`: Configuration file name for this host
- `profile_name`: WAS profile name
- `http_server`: HTTP server instance name

Example:
```yaml
ibmi-dev1:
  ansible_host: 10.1.2.101
  ansible_user: qsecofr
  ansible_ssh_pass: "{{ vault_dev1_password }}"
  config_file: dev1_deployment.config
  profile_name: WASPRODS
  http_server: WEBPRODS
```

### Configuration Files

Each configuration file maps to one host and contains:
- `WAS_REPO_PATH`: Path to WAS installation media
- `WAS_INSTALL_ROOT`: WAS product installation path
- `WAS_USER_ROOT`: WAS user data path
- `SAVE_LIB`: IBM i library for save files
- `PROFILE_SAVF`: Profile save file name
- `HTTP_DIR_SAVF`: HTTP directory save file name
- `HTTP_CFG_SAVF`: HTTP config save file name
- `PROFILE_NAME`: WAS profile name
- `HTTP_SERVER`: HTTP server instance name
- `APP_DIR`: Application directory
- `LOG_LEVEL`: INFO or DEBUG
- `DEFAULT_ACTION`: cleanup, retry, or abort
- `SKIP_INSTALL`: true or false

## Usage

### Basic Deployment

Deploy to development environment:
```bash
cd ansible
ansible-playbook playbooks/main.yml -e "target_env=dev"
```

Deploy to test environment:
```bash
ansible-playbook playbooks/main.yml -e "target_env=test"
```

Deploy to production environment:
```bash
ansible-playbook playbooks/main.yml -e "target_env=prod"
```

### Advanced Usage

**Deploy to specific host:**
```bash
ansible-playbook playbooks/main.yml -e "target_env=dev" --limit ibmi-dev1
```

**Skip save operation (use existing save files):**
```bash
ansible-playbook playbooks/main.yml -e "target_env=dev skip_save=true"
```

**Skip file transfer (files already on target systems):**
```bash
ansible-playbook playbooks/main.yml -e "target_env=dev skip_transfer=true"
```

**Run only deployment step:**
```bash
ansible-playbook playbooks/main.yml -e "target_env=dev skip_save=true skip_transfer=true"
```

**Dry run (check mode):**
```bash
ansible-playbook playbooks/main.yml -e "target_env=dev" --check
```

**Verbose output:**
```bash
ansible-playbook playbooks/main.yml -e "target_env=dev" -vvv
```

### Individual Playbook Execution

**Run only save operation:**
```bash
ansible-playbook playbooks/was_save.yml
```

**Run only file transfer:**
```bash
ansible-playbook playbooks/transfer_files.yml -e "target_group=dev"
```

**Run only deployment:**
```bash
ansible-playbook playbooks/was_deployment.yml -e "target_group=dev"
```

### Using Tags

**Run specific steps:**
```bash
# Run only save step
ansible-playbook playbooks/main.yml -e "target_env=dev" --tags save

# Run only transfer step
ansible-playbook playbooks/main.yml -e "target_env=dev" --tags transfer

# Run only deployment step
ansible-playbook playbooks/main.yml -e "target_env=dev" --tags deployment
```

## Workflow Details

### Step 1: WAS Save Operation

**Playbook:** `was_save.yml`  
**Target:** Template system (single host in `template_systems` group)

Actions:
1. Validates `was_save.sh` script exists
2. Validates template configuration file exists
3. Copies script and config to template system
4. Executes `was_save.sh -c template_save.config`
5. Verifies save files were created
6. Fetches operation logs

Output:
- Save files in specified library on template system
- Log file: `logs/ibmi-template_was_save_TIMESTAMP.log`

### Step 2: File Transfer

**Playbook:** `transfer_files.yml`  
**Target:** All hosts in specified environment group

Actions:
1. Creates target save file library if needed
2. Transfers save files from template system using FTP
3. Verifies transferred files exist
4. Validates file integrity

Output:
- Save files copied to each managed node
- Files in library specified in environment group_vars

### Step 3: WAS Deployment

**Playbook:** `was_deployment.yml`  
**Target:** All hosts in specified environment group  
**Execution:** Serial (one host at a time)

Actions:
1. Validates `was_full_deployment.sh` script exists
2. Validates host-specific configuration file exists
3. Copies script and config to managed node
4. Executes `was_full_deployment.sh -c <host_config_file>`
5. Verifies WAS profile and HTTP server are running
6. Fetches deployment logs and summary

Output:
- WAS profile deployed and running
- HTTP server configured and running
- Applications deployed
- Log files: `logs/<hostname>_was_deployment_TIMESTAMP.log`
- Summary: `logs/<hostname>_deployment_summary.txt`

## Logging

### Log Files

All logs are stored in the `logs/` directory:

- **Save logs**: `<template_host>_was_save_TIMESTAMP.log`
- **Deployment logs**: `<hostname>_was_deployment_TIMESTAMP.log`
- **Deployment summaries**: `<hostname>_deployment_summary.txt`
- **Ansible log**: `ansible.log` (in ansible directory)

### Log Levels

Configure log level in configuration files or group_vars:
- `INFO`: Basic operation logging
- `DEBUG`: Verbose logging with command outputs

## Error Handling

### Common Issues

**Issue: "Configuration file not found"**
- Verify config file exists in `configs/` directory
- Check `config_file` variable in inventory matches actual filename

**Issue: "Save files not found on template system"**
- Check template system save operation completed successfully
- Verify save file library and names in template_save.config
- Review template system logs

**Issue: "File transfer failed"**
- Verify network connectivity between systems
- Check FTP credentials and permissions
- Ensure target library exists and is accessible

**Issue: "Deployment failed"**
- Review deployment logs for specific errors
- Check save files were transferred successfully
- Verify WAS installation media path is correct
- Ensure sufficient disk space on target system

### Troubleshooting

**Enable debug logging:**
```bash
ansible-playbook playbooks/main.yml -e "target_env=dev log_level=DEBUG" -vvv
```

**Check connectivity:**
```bash
ansible all -m ping
ansible dev -m shell -a "system 'DSPLIBL'"
```

**Verify save files:**
```bash
ansible template_systems -m shell -a "system 'WRKOBJ OBJ(QGPL/WASPRODS) OBJTYPE(*FILE)'"
```

**Check deployment status:**
```bash
ansible dev -m shell -a "system 'WRKACTJOB JOB(WASPRODS)'"
```

## Security Considerations

### Credential Management

**Use Ansible Vault for passwords:**
```bash
# Create vault file
ansible-vault create inventory/vault.yml

# Encrypt individual strings
ansible-vault encrypt_string 'password' --name 'vault_dev1_password'

# Run playbook with vault
ansible-playbook playbooks/main.yml -e "target_env=dev" --ask-vault-pass
```

### File Permissions

```bash
# Secure inventory files
chmod 600 inventory/hosts.yml
chmod 600 inventory/vault.yml

# Secure configuration files
chmod 600 configs/*.config

# Secure ansible.cfg
chmod 600 ansible.cfg
```

### SSH Key Authentication

For better security, use SSH keys instead of passwords:

1. Generate SSH key pair on control node
2. Copy public key to IBM i systems
3. Remove `ansible_ssh_pass` from inventory
4. Add `ansible_ssh_private_key_file` to inventory

## Best Practices

1. **Test in Development First**
   - Always test deployments in dev environment
   - Verify all steps complete successfully
   - Review logs for warnings or errors

2. **Use Version Control**
   - Store inventory and configs in Git
   - Track changes to configuration files
   - Use branches for different environments

3. **Regular Backups**
   - Maintain backups of template system
   - Keep previous save files
   - Document configuration changes

4. **Monitor Deployments**
   - Review logs after each deployment
   - Verify applications are accessible
   - Check system performance

5. **Maintain Documentation**
   - Document environment-specific settings
   - Keep inventory up to date
   - Record deployment procedures

6. **Security**
   - Use Ansible Vault for sensitive data
   - Rotate passwords regularly
   - Limit access to control node
   - Use SSH keys when possible

## Maintenance

### Adding New Hosts

1. Add host to `inventory/hosts.yml` in appropriate group
2. Create configuration file in `configs/`
3. Set `config_file` variable for the host
4. Test connectivity: `ansible <hostname> -m ping`
5. Run deployment: `ansible-playbook playbooks/main.yml -e "target_env=<env>" --limit <hostname>`

### Updating Configuration

1. Edit configuration file in `configs/`
2. Commit changes to version control
3. Run deployment with updated config

### Removing Hosts

1. Remove host from `inventory/hosts.yml`
2. Archive or remove configuration file
3. Update documentation

## Support

For issues or questions:
- Review logs in `logs/` directory
- Check Ansible documentation: https://docs.ansible.com
- Review IBM i documentation
- Contact system administrators

## Related Documentation

- [was_save.sh README](../was_save/README.md) - WAS save script documentation
- [was_full_deployment.sh README](../was_install_deploy/README.md) - Deployment script documentation
- [Ansible IBM i Collection](https://galaxy.ansible.com/ibm/power_ibmi) - IBM i Ansible modules

## License

This project follows the same license as the parent WAS Deployment Automation project.

## Version History

- **v1.0.0** (03/18/2026) - Initial release
  - Main orchestration playbook
  - WAS save playbook
  - File transfer playbook
  - WAS deployment playbook
  - Environment-based inventory
  - 1:1 host-to-config mapping
  - Comprehensive logging and error handling

## Notes

- All scripts execute using `/usr/bin/sh` (IBM i Qshell)
- File transfers use native IBM i FTP in binary mode
- Deployments run serially to ensure proper monitoring
- Save files are created once and reused across deployments
- Each host must have a unique configuration file- Template system should have a fully configured WAS profile