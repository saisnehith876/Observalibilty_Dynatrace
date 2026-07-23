# Dynatrace OneAgent Installation using Ansible(Linux_Servers)

This guide walks through the complete process — from provisioning EC2 instances in AWS, setting up a master (control) node with Python and Ansible, verifying connectivity to target/slave nodes, and finally deploying Dynatrace OneAgent to all target nodes via a custom Ansible playbook.

---

## Architecture

```
                ┌─────────────────────┐
                │   Master Node (EC2)  │
                │  - Python            │
                │  - Ansible           │
                │  - .pem key          │
                └──────────┬───────────┘
                            │ SSH (port 22)
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
 ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
 │ Target Node 1│    │ Target Node 2│    │ Target Node N│
 │  (RHEL/Ubuntu)│    │ (RHEL/Ubuntu)│    │ (RHEL/Ubuntu)│
 │  OneAgent    │    │  OneAgent    │    │  OneAgent    │
 └─────────────┘    └─────────────┘    └─────────────┘
```

---

## Prerequisites

- An AWS account with permissions to create EC2 instances, security groups, and key pairs
- A Dynatrace environment URL and a **PaaS token** with `InstallerDownload` scope
- Basic familiarity with SSH

---

## Step 1: Launch EC2 Instances (Master + Target Nodes)

1. Log in to the **AWS Console** → **EC2** → **Launch Instance**.
2. Launch **one master node** and **one or more target/slave nodes**:
   - AMI: Amazon Linux 2 / Ubuntu 22.04 (mix is fine, since the playbook supports both)
   - Instance type: `t2.micro` (free tier) or as needed
   - Key pair: create a new key pair (e.g. `dynatrace-key.pem`) and download it — **use the same key pair for master and all target nodes** so the master can SSH into them
   - Network settings: same VPC/subnet for all instances so they can reach each other privately
   - Security group: allow inbound
     - SSH (port 22) — from your IP (for you to reach the master) and from the master's security group/private IP range (so master can reach targets)
     - Custom TCP outbound to Dynatrace cluster on port 443 (usually allowed by default via outbound rules)
3. Launch the instances. Note down the **public IP** of the master node and the **private IPs** of all target nodes (master will connect to targets over the private network within the VPC).

---

## Step 2: Log Into the Master Node

From your local machine:

```bash
chmod 400 dynatrace-key.pem
ssh -i dynatrace-key.pem ec2-user@<master-public-ip>      # Amazon Linux
# or
ssh -i dynatrace-key.pem ubuntu@<master-public-ip>        # Ubuntu
```

---

## Step 3: Install Python and Ansible on the Master Node

### Amazon Linux / RHEL:

```bash
sudo yum update -y
sudo yum install -y python3 python3-pip
sudo amazon-linux-extras install ansible2 -y
# or, if not available via extras:
sudo pip3 install ansible

ansible --version
python3 --version
```

### Ubuntu/Debian:

```bash
sudo apt update -y
sudo apt install -y python3 python3-pip
sudo apt install -y ansible

ansible --version
python3 --version
```

---

## Step 4: Copy the PEM Key to the Master Node and Set Permissions

The master node needs the same `.pem` key to SSH into the target nodes.

From your **local machine**, copy the key to the master:

```bash
scp -i dynatrace-key.pem dynatrace-key.pem ec2-user@<master-public-ip>:~/
```

Then, back **on the master node**, restrict the key's permissions (SSH refuses to use keys that are too open):

```bash
chmod 400 ~/dynatrace-key.pem
```

---

## Step 5: Set Up the Ansible Inventory on the Master Node

Create a working directory and inventory file:

```bash
mkdir -p ~/dynatrace-ansible
cd ~/dynatrace-ansible
nano inventory.ini
```

`inventory.ini`:

```ini
[target_nodes]
target1 ansible_host=<target1-private-ip> ansible_user=ec2-user
target2 ansible_host=<target2-private-ip> ansible_user=ubuntu
target3 ansible_host=<target3-private-ip> ansible_user=ec2-user

[target_nodes:vars]
ansible_ssh_private_key_file=~/dynatrace-key.pem
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

> Use `ansible_user=ec2-user` for Amazon Linux/RHEL targets and `ansible_user=ubuntu` for Ubuntu targets.

---

## Step 6: Verify Connectivity to Target Nodes (Ping Test)

Test raw SSH connectivity first:

```bash
ssh -i ~/dynatrace-key.pem ec2-user@<target1-private-ip>
exit
```

Then use Ansible's built-in `ping` module to confirm all targets respond:

```bash
ansible -i inventory.ini target_nodes -m ping
```

Expected output for each host:

```
target1 | SUCCESS => {
    "ansible_facts": { "discovered_interpreter_python": "/usr/bin/python3" },
    "changed": false,
    "ping": "pong"
}
```

If a host fails, check:
- Security group allows SSH from the master's private IP
- Correct `ansible_user` for that OS
- Key permissions (`chmod 400`)

---

## Step 7: Store the Dynatrace PaaS Token and Environment URL

Use Ansible Vault so the token isn't stored in plaintext:

```bash
ansible-vault create group_vars/all/vault.yml
```

Inside `vault.yml`:

```yaml
dynatrace_environment_url: "https://<environment-id>.live.dynatrace.com"
dynatrace_paas_token: "dt0c01.XXXXXXXX...XXXXXXXX"
```

---

## Step 8: Create the OneAgent Installation Playbook

`install_oneagent.yml`:

```yaml
---
- name: Install Dynatrace OneAgent on all target nodes
  hosts: target_nodes
  become: true
  vars_files:
    - group_vars/all/vault.yml
  vars:
    oneagent_install_path: "/opt/dynatrace/oneagent"
    oneagent_download_dir: "/tmp/dynatrace-installer"
    oneagent_host_group: "linux-prod"

  tasks:
    - name: Ensure download directory exists
      ansible.builtin.file:
        path: "{{ oneagent_download_dir }}"
        state: directory
        mode: "0750"

    - name: Check if OneAgent is already installed
      ansible.builtin.stat:
        path: "{{ oneagent_install_path }}/agent/bin/oneagentctl"
      register: oneagent_installed

    - name: Download OneAgent installer via Dynatrace API
      ansible.builtin.get_url:
        url: "{{ dynatrace_environment_url }}/api/v1/deployment/installer/agent/unix/default/latest?arch=x86_64&flavor=default"
        dest: "{{ oneagent_download_dir }}/Dynatrace-OneAgent-Linux.sh"
        headers:
          Authorization: "Api-Token {{ dynatrace_paas_token }}"
        mode: "0750"
        timeout: 120
      when: not oneagent_installed.stat.exists

    - name: Install OneAgent
      ansible.builtin.command: >
        /bin/sh {{ oneagent_download_dir }}/Dynatrace-OneAgent-Linux.sh
        --set-host-group={{ oneagent_host_group }}
        --set-app-log-content-access=true
      when: not oneagent_installed.stat.exists
      register: oneagent_install_result

    - name: Ensure OneAgent service is running
      ansible.builtin.systemd:
        name: oneagent
        state: started
        enabled: true

    - name: Confirm OneAgent version
      ansible.builtin.command: "{{ oneagent_install_path }}/agent/bin/oneagentctl --version"
      register: oneagent_version
      changed_when: false

    - name: Display installed version
      ansible.builtin.debug:
        msg: "OneAgent installed on {{ inventory_hostname }}: {{ oneagent_version.stdout }}"

    - name: Clean up installer file
      ansible.builtin.file:
        path: "{{ oneagent_download_dir }}/Dynatrace-OneAgent-Linux.sh"
        state: absent
```

---

## Step 9: Run the Playbook

Dry run first:

```bash
ansible-playbook -i inventory.ini install_oneagent.yml --ask-vault-pass --check
```

Actual run across all target nodes:

```bash
ansible-playbook -i inventory.ini install_oneagent.yml --ask-vault-pass
```

Run against a single node only (for testing):

```bash
ansible-playbook -i inventory.ini install_oneagent.yml --ask-vault-pass --limit target1
```

---

## Step 10: Verify Installation on Target Nodes

SSH into any target node (or use `ansible ... -m shell`) and check:

```bash
sudo systemctl status oneagent
sudo /opt/dynatrace/oneagent/agent/bin/oneagentctl --version
sudo /opt/dynatrace/oneagent/agent/bin/oneagentctl --show-hostname
```

Or check all nodes at once from the master:

```bash
ansible -i inventory.ini target_nodes -m shell -a "systemctl is-active oneagent" --become
```

Then confirm in the **Dynatrace UI** → **Hosts** that each node appears with "OneAgent monitoring active" (allow 2–5 minutes for reporting).

---

## Troubleshooting

| Issue | Likely Cause | Fix |
|---|---|---|
| `ansible ping` fails / UNREACHABLE | Security group blocking SSH from master, wrong key, or wrong `ansible_user` | Check SG inbound rules on target, confirm key path/permissions, confirm OS-specific username |
| `Permission denied (publickey)` | PEM key not copied to master or wrong permissions | Re-`scp` the key, run `chmod 400` on the master |
| `401 Unauthorized` on installer download | Invalid/expired PaaS token | Regenerate token in Dynatrace UI with `InstallerDownload` scope |
| Installer download times out | No outbound 443 from target to Dynatrace | Check target's security group/NACL allows outbound HTTPS |
| Host not appearing in Dynatrace UI | Wrong environment URL or firewall blocking OneAgent traffic | Verify `dynatrace_environment_url`, allow outbound 443 |
| `oneagentctl: command not found` | Install failed silently | Check `oneagent_install_result.stdout/stderr`, review `/var/log/dynatrace/oneagent/` |

---

## Security Notes

- Never commit the `.pem` key or `vault.yml` to version control.
- Restrict the master node's security group so only your IP can SSH into it.
- Restrict target nodes' security groups so only the master's private IP can SSH into them.
- Use `ansible-vault` for the PaaS token; rotate tokens periodically.
- Scope the PaaS token to `InstallerDownload` only.

---

## Quick Reference — Full Command Sequence

```bash
# Local machine
chmod 400 dynatrace-key.pem
ssh -i dynatrace-key.pem ec2-user@<master-public-ip>

# On master node
sudo yum install -y python3 python3-pip && sudo amazon-linux-extras install ansible2 -y
scp -i dynatrace-key.pem dynatrace-key.pem ec2-user@<master-public-ip>:~/   # (run from local machine)
chmod 400 ~/dynatrace-key.pem                                              # (run on master)

# On master node
ansible -i inventory.ini target_nodes -m ping
ansible-vault create group_vars/all/vault.yml
ansible-playbook -i inventory.ini install_oneagent.yml --ask-vault-pass
```
