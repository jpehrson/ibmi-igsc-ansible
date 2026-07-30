# ibmi-igsc-ansible

Ansible playbooks for automating IBM i system administration tasks, with a focus on Electronic Service Agent (ESA) lifecycle management and system service operations.

## Prerequisites

- **Ansible** 2.9 or later
- **IBM Power IBM i collection**: `ibm.power_ibmi`
  ```bash
  ansible-galaxy collection install ibm.power_ibmi
  ```
- **IBM i hosts** with SSH enabled and Python 3 available at `/QOpenSys/pkgs/bin/python3.9`
- `*SECOFR` authority on managed IBM i systems for ESA and system operations

## Repository Structure

```
ibmi-igsc-ansible/
├── inventory/
│   ├── hosts.yml                   # Host definitions (dev, test, prod, template_systems)
│   └── group_vars/
│       ├── all.yml                 # Variables common to all hosts
│       ├── dev.yml                 # Development environment variables
│       ├── test.yml                # Test environment variables
│       ├── prod.yml                # Production environment variables
│       └── template_systems.yml   # Template system variables
└── playbooks/
    ├── Utility playbooks           # QMGTOOLS, SSH keys, PTF checks, SQL samples
    ├── System service playbooks    # Subsystem and server control
    └── ESA lifecycle playbooks     # Electronic Service Agent management
```

## Inventory

The inventory in [`inventory/hosts.yml`](inventory/hosts.yml) organizes hosts into four groups:

| Group | Purpose |
|---|---|
| `template_systems` | Source system for creating WAS save files |
| `dev` | Development environment hosts |
| `test` | Test environment hosts |
| `prod` | Production environment hosts |

Host passwords for `dev` and `test` environments are stored as Ansible Vault-encrypted strings in their respective group_vars files. Production hosts use vault-encrypted values that must be supplied before running playbooks against `prod`.

To target a specific environment or host:
```bash
ansible-playbook playbooks/<playbook>.yml -e "target_env=dev"
ansible-playbook playbooks/<playbook>.yml --limit ibmi-dev1
```

## Playbooks

### Utilities

#### [`ibmi-install-qmgtools.yml`](playbooks/ibmi-install-qmgtools.yml)
Downloads the current version of QMGTOOLS from IBM's public site, copies it into `QGPL`, and restores it to the `QMGTOOLS` library. The save file URL is built automatically from the target system's OS version. Targets the `ibmi` host group.

```bash
ansible-playbook playbooks/ibmi-install-qmgtools.yml
```

---

#### [`deploy_ssh_keys.yml`](playbooks/deploy_ssh_keys.yml)
Deploys SSH public keys from a local `ssh_keys/` directory to managed hosts' `~/.ssh/authorized_keys`. Expects key files named `<inventory_hostname>_authorized_keys`.

```bash
ansible-playbook playbooks/deploy_ssh_keys.yml --limit vk4060 --ask-pass
ansible-playbook playbooks/deploy_ssh_keys.yml -e "target_env=test" --ask-pass
```

---

#### [`ibmi-fix-check.yml`](playbooks/ibmi-fix-check.yml)
Checks the status of individual PTFs and PTF groups on the target IBM i system using the `ibmi_fix_check` module. Targets `vk4060` directly.

```bash
ansible-playbook playbooks/ibmi-fix-check.yml
```

---

#### [`ibmi-sql-sample.yml`](playbooks/ibmi-sql-sample.yml)
Demonstrates SQL execution on IBM i: creates a journal and receiver, creates a table in `QGPL`, inserts rows, queries results, and cleans up. Useful as a template for SQL-based playbooks. Targets `vk4060`.

```bash
ansible-playbook playbooks/ibmi-sql-sample.yml
```

---

### System Services

#### [`ibmi_start_sbs__host_and_tcp_servers.yml`](playbooks/ibmi_start_sbs__host_and_tcp_servers.yml)
Starts IBM i subsystems (`QSYSWRK`, `QSERVER`, `QUSRWRK`, `QBATCH`, `QSPL`), all host servers, and autostart TCP services.

```bash
ansible-playbook playbooks/ibmi_start_sbs__host_and_tcp_servers.yml
```

---

#### [`ibmi_endsbs_all_except_SSH.yml`](playbooks/ibmi_endsbs_all_except_SSH.yml)
Stops all IBM i host servers, a broad set of TCP services, and multiple subsystems immediately, leaving only SSH and Ansible connectivity intact. Also stops the IBM i Debug Service.

```bash
ansible-playbook playbooks/ibmi_endsbs_all_except_SSH.yml
```

---

#### [`ibmi_endsbs_all_except_SSH_Telnet.yml`](playbooks/ibmi_endsbs_all_except_SSH_Telnet.yml)
Same as above but preserves Telnet connectivity in addition to SSH by omitting `QCMN` and `QSYSWRK` from the subsystem shutdown list.

```bash
ansible-playbook playbooks/ibmi_endsbs_all_except_SSH_Telnet.yml
```

---

#### [`ibmi_end_DTAQ.yaml`](playbooks/ibmi_end_DTAQ.yaml)
Stops only the `*DTAQ` host server.

```bash
ansible-playbook playbooks/ibmi_end_DTAQ.yaml
```

---

### Electronic Service Agent (ESA)

These playbooks automate the full ESA configuration lifecycle on IBM i. Run them in order when setting up ESA from scratch.

#### [`ibmi-esa-create.yml`](playbooks/ibmi-esa-create.yml)
Creates the ESA service configuration using `CRTSRVCFG`. Validates that ESA is not already configured and supports optional proxy configuration.

```bash
ansible-playbook playbooks/ibmi-esa-create.yml --limit vk4060
ansible-playbook playbooks/ibmi-esa-create.yml --limit vk4060 --check
ansible-playbook playbooks/ibmi-esa-create.yml -e "esa_use_proxy=true esa_proxy_server=proxy.example.com esa_proxy_port=8080"
```

Key variables:

| Variable | Default | Description |
|---|---|---|
| `esa_cfg_type` | `*DIRECT` | Connection type (`*DIRECT` or `*VPN`) |
| `esa_country_code` | `*SELECT` | IBM i country code (e.g., `US`, `DE`) |
| `esa_state_code` | `*SELECT` | State/region code |
| `esa_use_proxy` | `false` | Enable proxy server |
| `esa_proxy_server` | `''` | Proxy hostname or IP |
| `esa_proxy_port` | `''` | Proxy port |

---

#### [`ibmi-esa-configure.yml`](playbooks/ibmi-esa-configure.yml)
Configures ESA customer and contact information using `CHGSRVAGTA`. Requires ESA to be created first.

```bash
ansible-playbook playbooks/ibmi-esa-configure.yml --limit vk4060
ansible-playbook playbooks/ibmi-esa-configure.yml --limit vk4060 --check
```

Required variables (set in `host_vars` or passed with `-e`):

| Variable | Description |
|---|---|
| `esa_contact_name` | Primary contact full name |
| `esa_contact_phone` | Primary contact phone number |
| `esa_contact_email` | Primary contact email address |

Optional variables: `esa_customer_number`, `esa_customer_name`, `esa_customer_country`, `esa_system_contact_*`, `esa_alternate_contact_*`, `esa_collect_time`, `esa_transmission_schedule`.

---

#### [`ibmi-esa-enable.yml`](playbooks/ibmi-esa-enable.yml)
Enables ESA using `CHGSRVAGTA ENABLE(*YES)` and verifies that all four required jobs start in the `QSYSWRK` subsystem.

```bash
ansible-playbook playbooks/ibmi-esa-enable.yml --limit vk4060
```

Expected jobs after enable: `QS9HDWMON`, `QS9PALMON`, `QS9PRBMON`, `QS9PRBSND`

---

#### [`ibmi-esa-reconfigure.yml`](playbooks/ibmi-esa-reconfigure.yml)
Full reconfiguration workflow: optionally deletes the existing ESA configuration and recreates it. Useful for correcting misconfigured systems.

```bash
ansible-playbook playbooks/ibmi-esa-reconfigure.yml --limit vk4060
ansible-playbook playbooks/ibmi-esa-reconfigure.yml --limit vk4060 --check
```

---

#### [`ibmi-esa-delete.yml`](playbooks/ibmi-esa-delete.yml)
Deletes the ESA configuration following IBM's official procedure:
1. `CHGSRVAGTA ENABLE(*NO)` — disable ESA
2. `DLTSRVCFG` — delete service configuration
3. Remove PPP and SDLC line descriptions starting with `Q`
4. Remove `/QIBM/UserData/OS400/UniversalConnection/*`
5. Remove `/QIBM/UserData/OS400/ServiceAgent/Contact.properties`

Creates a local backup of the current ESA configuration before deletion by default.

```bash
ansible-playbook playbooks/ibmi-esa-delete.yml --limit vk4060
ansible-playbook playbooks/ibmi-esa-delete.yml --limit vk4060 --check
```

---

#### [`ibmi-esa-delete-test.yml`](playbooks/ibmi-esa-delete-test.yml)
**Read-only** diagnostic playbook. Queries what objects would be affected by a deletion versus what IBM documentation says should be deleted. Run this before `ibmi-esa-delete.yml` to verify the scope of a deletion.

```bash
ansible-playbook playbooks/ibmi-esa-delete-test.yml --limit vk4060
```

---

#### [`ibmi-esa-verify-config.yml`](playbooks/ibmi-esa-verify-config.yml)
Comprehensive ESA verification: checks configuration completeness, active jobs, last transmission status, recent problem log, and network connectivity to `esupport.ibm.com`.

```bash
ansible-playbook playbooks/ibmi-esa-verify-config.yml --limit vk4060
ansible-playbook playbooks/ibmi-esa-verify-config.yml --limit vk4060 -e "fail_on_errors=true"
ansible-playbook playbooks/ibmi-esa-verify-config.yml -e "days_back=7"
```

---

#### [`ibmi-esa-verify-jobs.yml`](playbooks/ibmi-esa-verify-jobs.yml)
Checks that all required ESA jobs (`QS9HDWMON`, `QS9PALMON`, `QS9PRBMON`, `QS9PRBSND`) are active in the `QSYSWRK` subsystem. Also checks the optional `QS9UAK` job for standalone Power 8+ partitions.

```bash
ansible-playbook playbooks/ibmi-esa-verify-jobs.yml --limit vk4060
ansible-playbook playbooks/ibmi-esa-verify-jobs.yml --limit vk4060 -e "fail_on_missing_jobs=true"
```

---

#### [`ibmi-esa-info.yml`](playbooks/ibmi-esa-info.yml)
Queries `QSYS2.ELECTRONIC_SERVICE_AGENT_INFO` to display ESA connection status, server configuration, and proxy settings.

```bash
ansible-playbook playbooks/ibmi-esa-info.yml --limit vk4060
ansible-playbook playbooks/ibmi-esa-info.yml -e "output_format=json"
ansible-playbook playbooks/ibmi-esa-info.yml -e "save_to_file=true"
```

---

#### [`ibmi-esa-diagnose.yml`](playbooks/ibmi-esa-diagnose.yml)
Runs nine diagnostic checks to explain why ESA jobs may not be running: configuration status, active jobs, job schedule entries, recent job history, contact information, network connectivity, relevant system values, QSRVAGT library objects, and QSYSOPR messages. Saves a report to `/tmp/` on the Ansible controller.

```bash
ansible-playbook playbooks/ibmi-esa-diagnose.yml --limit vk4060
```

---

#### [`ibmi-esa-discover-columns.yml`](playbooks/ibmi-esa-discover-columns.yml)
Queries `QSYS2.SYSCOLUMNS` to list the columns available in `QSYS2.ELECTRONIC_SERVICE_AGENT_INFO` on the target system. Useful when column availability varies by IBM i version or PTF level.

```bash
ansible-playbook playbooks/ibmi-esa-discover-columns.yml --limit vk4060
```

---

#### [`esa_refresh_config.yml`](playbooks/esa_refresh_config.yml)
Deletes and recreates the ESA service configuration using the system country ID (`QCNTRYID` system value). Targets the `ibmi` host group.

```bash
ansible-playbook playbooks/esa_refresh_config.yml
```

---

## ESA Setup Workflow

To configure ESA end-to-end on a system that has no existing configuration:

```bash
# 1. Create the ESA service configuration
ansible-playbook playbooks/ibmi-esa-create.yml --limit <host>

# 2. Set customer and contact information
ansible-playbook playbooks/ibmi-esa-configure.yml --limit <host> \
  -e "esa_contact_name='IT Operations' esa_contact_phone='+1-555-0100' esa_contact_email='ops@example.com'"

# 3. Enable ESA
ansible-playbook playbooks/ibmi-esa-enable.yml --limit <host>

# 4. Verify jobs and configuration
ansible-playbook playbooks/ibmi-esa-verify-jobs.yml --limit <host>
ansible-playbook playbooks/ibmi-esa-verify-config.yml --limit <host>
```

## Secrets Management

Host passwords are encrypted with Ansible Vault. To encrypt a new password:

```bash
ansible-vault encrypt_string 'your_password' --name 'vault_dev1_password'
```

Paste the output into the appropriate `inventory/group_vars/<env>.yml` file. When running playbooks, supply the vault password:

```bash
ansible-playbook playbooks/<playbook>.yml --ask-vault-pass
# or
ansible-playbook playbooks/<playbook>.yml --vault-password-file ~/.vault_pass
```

## License

Apache License 2.0 — see [LICENSE](LICENSE) for details.
