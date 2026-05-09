# jumphost

> Part of [dev-hub/Ansible](https://github.com/CollinPoetoehena/dev-hub/blob/main/Ansible.md) — see that file for conventions, structure guidelines, and the full role index.

Configures the bastion/jump host VM that serves as the secure entry point to the infrastructure. This role sets up SSH forwarding, access controls, and security hardening to provide safe gateway access to the private Kubernetes cluster network.

It has the following features:
- **SSH Forwarding**: Enable and configure SSH agent forwarding for secure access
- **Bastion Security**: Additional hardening specific to bastion host requirements
- **Access Logging**: Configure audit logging for all SSH connections
- **Firewall Rules**: Restrict inbound/outbound traffic to essential SSH connections only
- **Minimal Attack Surface**: Keep installed packages and services to a minimum to reduce potential vulnerabilities

## Requirements

- Ansible 2.9 or higher
- Target host running Enterprise Linux (RHEL, CentOS, AlmaLinux, Rocky Linux, etc.)
- The `ansible.posix` collection: `ansible-galaxy collection install ansible.posix`
- The `azureuser` account (or equivalent `jump_host_user`) must exist on the target host

## Variables

- You must define the variable `ssh_allowed_ip` (e.g., in your inventory, group_vars, or vault) in your project. This role does not set or store this value. **Important:**
  - Do **not** define or commit this variable in the role itself.
  - The role expects `ssh_allowed_ip` to be set by the playbook or inventory that includes this role.
  - This keeps sensitive information out of the role and under your project's control (ssh).

| Variable | Default | Description |
|----------|---------|-------------|
| `jump_host_user` | `"azureuser"` | OS user account on the jump host |
| `jump_host_ssh_keys_path` | `"/home/{{ jump_host_user }}/.ssh"` | Path to the SSH keys directory on the jump host |
| `enable_ssh_agent_forwarding` | `true` | Enable SSH agent forwarding (required for Ansible to reach cluster nodes via jump host) |
| `ssh_allowed_users` | `["{{ jump_host_user }}"]` | List of OS users permitted to log in via SSH; empty list allows all users |
| `ssh_max_sessions` | `10` | Maximum number of concurrent SSH sessions |
| `ssh_login_grace_time` | `60` | Seconds before an unauthenticated connection is dropped |
| `ssh_max_auth_tries` | `3` | Maximum authentication attempts per connection |
| `ssh_client_alive_interval` | `300` | Keepalive interval in seconds; idle sessions are disconnected after `interval × count_max` |
| `ssh_client_alive_count_max` | `2` | Number of unanswered keepalives before the session is terminated |
| `enable_ssh_audit_logging` | `true` | Enable dedicated audit logging for SSH connections |
| `audit_log_path` | `"/var/log/ssh-audit.log"` | Path to the SSH audit log file (rotated daily, kept for 30 days) |

## Usage

Requirements file example (same directory as ansible.cfg, create a file called requirements.yml):
```yaml
---
roles:
  - name: jumphost
    src: https://github.com/CollinPoetoehena/ansible-role-jumphost.git
    scm: git
    version: v0.0.1
``` 

Then install with: 
```sh
# NOTE: Example of roles path for -p is "roles/" (you can also specify this in ansible.cfg)
ansible-galaxy install -r requirements.yml -p <path/to/roles>
```

Example playbook using this role (e.g. site.yml):
```yaml
- hosts: all
  roles:
    - role: jumphost
```
