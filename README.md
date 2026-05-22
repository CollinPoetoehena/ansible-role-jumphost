# devhub-ansible-jumphost

> Part of [DevHub/Ansible](https://github.com/CollinPoetoehena/DevHub/blob/main/packages/Ansible.md) — see that file for conventions, structure guidelines, and the full role index.

Configures the bastion/jump host VM that serves as the secure entry point to the infrastructure. This role sets up SSH forwarding, access controls, and security hardening to provide safe gateway access to the private Kubernetes cluster network.

It has the following features:
- **SSH Forwarding**: Enable and configure SSH agent forwarding for secure access
- **Bastion Security**: Additional hardening specific to bastion host requirements
- **Access Logging**: Configure audit logging for all SSH connections
- **Firewall Rules**: Restrict inbound/outbound traffic to essential SSH connections only
- **Minimal Attack Surface**: Keep installed packages and services to a minimum to reduce potential vulnerabilities

> **User, group, sudo, and SSH key management are not part of this role.** Use the [devhub-ansible-users](https://github.com/CollinPoetoehena/devhub-ansible-users) role for that. Apply it alongside this role in your playbook (see example below).

## Requirements

- **OS**: Any Linux distribution (RHEL/EL, Debian/Ubuntu, etc.), uses `ansible.builtin.package` for package management to support multiple distributions.
- **Ansible**: 2.14+
- **Collections**: `ansible.posix`
  Install with:
  ```sh
  ansible-galaxy collection install ansible.posix
  ```

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
  - name: devhub.jumphost
    src: https://github.com/CollinPoetoehena/devhub-ansible-jumphost.git
    scm: git
    version: 1.0.0
  # User/group/sudo/SSH key management is handled by a separate role:
  - name: devhub.users
    src: https://github.com/CollinPoetoehena/devhub-ansible-users.git
    scm: git
    version: 1.0.0
```

Then install with:
```sh
# NOTE: Example of roles path for -p is "roles/" (you can also specify this in ansible.cfg)
ansible-galaxy install -r requirements.yml -p <path/to/roles>
```

Example playbook using this role (e.g. site.yml):

> **Note:** User, group, sudo, and SSH key management are handled by [devhub-ansible-users](https://github.com/CollinPoetoehena/devhub-ansible-users).  
> Variables and usage examples for that role are intentionally omitted here — keeping them in one place avoids duplication and means only that role's README needs updating if its interface changes.

```yaml
---
- hosts: jumphost
  become: true
  vars:
    jump_host_user: azureuser
    ssh_allowed_ip: "{{ vault_ssh_allowed_ip }}"  # load from vault

  roles:
    - role: devhub.jumphost  # hardens SSH, firewall, audit logging
    - role: devhub.users     # manages OS users, groups, sudo, and SSH keys (see devhub-ansible-users)
```

### Example 
Below is an example demonstrating how the role operates in practice, including how you can verify and present proof of SSH audit logging in action. This shows what you would see after applying the role, both for validation/testing of the role (i.e. showing the expected behavior) and for documentation purposes:

Initial rollout without reboot (note that it calls site.yml which was just an example playbook that includes the jumphost role, the content of site.yml is not important here, just that it includes the jumphost role and is run with the appropriate vault password file to access the ssh_allowed_ip variable):
```sh
ansible-playbook -i hosts site.yml --tags jumphost --vault-password-file ~/.vault_pass.txt

PLAY [Configure Jump Host (Bastion)] *******************************************************************************************************************************************************************

TASK [jumphost : Remove unnecessary packages] **********************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Gather service facts] *****************************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Disable unnecessary services if present] **********************************************************************************************************************************************
skipping: [20.127.29.83] => (item=avahi-daemon) 
skipping: [20.127.29.83] => (item=cups) 
skipping: [20.127.29.83] => (item=rpcbind) 
skipping: [20.127.29.83] => (item=nfs-server) 
skipping: [20.127.29.83]

TASK [jumphost : Ensure .ssh directory exists with correct permissions] ********************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Deploy hardened sshd_config (the configuration used by SSH daemon)] *******************************************************************************************************************
changed: [20.127.29.83]

TASK [jumphost : Ensure sshd is enabled and started] ***************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Ensure firewalld is installed] ********************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Ensure firewalld is enabled and started] **********************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Allow SSH only from specific IP] ******************************************************************************************************************************************************
changed: [20.127.29.83]

TASK [jumphost : Add rate limiting] ********************************************************************************************************************************************************************
changed: [20.127.29.83]

TASK [jumphost : Configure audit logging] **************************************************************************************************************************************************************
included: /home/poetoec/projects/personal/k8s-lab/ansible/roles/jumphost/tasks/audit_logging.yml for 20.127.29.83

TASK [jumphost : Ensure audit and rsyslog packages are installed] **************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Ensure rsyslog is running] ************************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Ensure auditd is running] *************************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Deploy audit rules] *******************************************************************************************************************************************************************
changed: [20.127.29.83]

TASK [jumphost : Flush handlers] ***********************************************************************************************************************************************************************

RUNNING HANDLER [jumphost : Restart sshd] **************************************************************************************************************************************************************
changed: [20.127.29.83]

RUNNING HANDLER [jumphost : Reload audit rules] ********************************************************************************************************************************************************
changed: [20.127.29.83]

TASK [jumphost : Reboot to apply new audit rules if immutable mode was active] *************************************************************************************************************************
skipping: [20.127.29.83]

TASK [jumphost : Ensure SSH audit log directory exists] ************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Ensure SSH audit log file exists] *****************************************************************************************************************************************************
changed: [20.127.29.83]

TASK [jumphost : Deploy rsyslog config to forward SSH logs to dedicated audit file] ********************************************************************************************************************
changed: [20.127.29.83]

RUNNING HANDLER [jumphost : Restart rsyslog] ***********************************************************************************************************************************************************
changed: [20.127.29.83]

TASK [Display jump host information] *******************************************************************************************************************************************************************
ok: [20.127.29.83] => {
    "msg": [
        "Jump host (bastion) configuration completed successfully",
        "SSH access gateway is ready",
        "Use this host to securely access the mgmtvm"
    ]
}

PLAY RECAP *********************************************************************************************************************************************************************************************
20.127.29.83               : ok=21   changed=9    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
```

Verification of SSH audit logging:
```sh
[azureuser@jumphost ~]$ sudo su
[root@jumphost azureuser]# cat /var/log/ssh-audit.log 
May 14 14:34:16 jumphost sshd[7047]: Connection from 77.165.126.69 port 55856 on 10.10.1.4 port 22 rdomain ""
May 14 14:34:17 jumphost sshd[7047]: Accepted key RSA SHA256:ROG4Um0kw0bEc4i7rn6G0zhQ6i84gUGliHOPwTjnZl4 found at /home/azureuser/.ssh/authorized_keys:1
May 14 14:34:17 jumphost sshd[7047]: Postponed publickey for azureuser from 77.165.126.69 port 55856 ssh2 [preauth]
May 14 14:34:17 jumphost sshd[7047]: Accepted key RSA SHA256:ROG4Um0kw0bEc4i7rn6G0zhQ6i84gUGliHOPwTjnZl4 found at /home/azureuser/.ssh/authorized_keys:1
May 14 14:34:17 jumphost sshd[7047]: Accepted publickey for azureuser from 77.165.126.69 port 55856 ssh2: RSA SHA256:ROG4Um0kw0bEc4i7rn6G0zhQ6i84gUGliHOPwTjnZl4
May 14 14:34:17 jumphost sshd[7047]: pam_unix(sshd:session): session opened for user azureuser(uid=1000) by azureuser(uid=0)
May 14 14:34:17 jumphost sshd[7047]: User child is on pid 7050
May 14 14:34:17 jumphost sshd[7050]: Starting session: shell on pts/0 for azureuser from 77.165.126.69 port 55856 id 0
```

Final verification: Changing rules in audit_logging.yml and rerunning the role to trigger the reboot task if immutable mode is active (using the same command and setup as before to run the playbook):
```sh
ansible-playbook -i hosts site.yml --tags jumphost --vault-password-file ~/.vault_pass.txt

PLAY [Configure Jump Host (Bastion)] *******************************************************************************************************************************************************************

TASK [jumphost : Remove unnecessary packages] **********************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Gather service facts] *****************************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Disable unnecessary services if present] **********************************************************************************************************************************************
skipping: [20.127.29.83] => (item=avahi-daemon) 
skipping: [20.127.29.83] => (item=cups) 
skipping: [20.127.29.83] => (item=rpcbind) 
skipping: [20.127.29.83] => (item=nfs-server) 
skipping: [20.127.29.83]

TASK [jumphost : Ensure .ssh directory exists with correct permissions] ********************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Deploy hardened sshd_config (the configuration used by SSH daemon)] *******************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Ensure sshd is enabled and started] ***************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Ensure firewalld is installed] ********************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Ensure firewalld is enabled and started] **********************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Allow SSH only from specific IP] ******************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Add rate limiting] ********************************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Configure audit logging] **************************************************************************************************************************************************************
included: /home/poetoec/projects/personal/k8s-lab/ansible/roles/jumphost/tasks/audit_logging.yml for 20.127.29.83

TASK [jumphost : Ensure audit and rsyslog packages are installed] **************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Ensure rsyslog is running] ************************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Ensure auditd is running] *************************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Deploy audit rules] *******************************************************************************************************************************************************************
changed: [20.127.29.83]

TASK [jumphost : Flush handlers] ***********************************************************************************************************************************************************************

RUNNING HANDLER [jumphost : Reload audit rules] ********************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Reboot to apply new audit rules if immutable mode was active] *************************************************************************************************************************
changed: [20.127.29.83]

TASK [jumphost : Ensure SSH audit log directory exists] ************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Ensure SSH audit log file exists] *****************************************************************************************************************************************************
ok: [20.127.29.83]

TASK [jumphost : Deploy rsyslog config to forward SSH logs to dedicated audit file] ********************************************************************************************************************
ok: [20.127.29.83]

TASK [Display jump host information] *******************************************************************************************************************************************************************
ok: [20.127.29.83] => {
    "msg": [
        "Jump host (bastion) configuration completed successfully",
        "SSH access gateway is ready",
        "Use this host to securely access the mgmtvm"
    ]
}

PLAY RECAP *********************************************************************************************************************************************************************************************
20.127.29.83               : ok=20   changed=2    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
```
