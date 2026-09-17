# RHEL Automation & Infrastructure Lab

A hands-on Linux systems administration and infrastructure automation project built using **Rocky Linux, Ansible, Bash, Podman, Firewalld, Apache HTTP Server, SSH, and Git**.

The project simulates a small production-style Linux environment with a dedicated Ansible control node managing multiple Linux servers. The objective is to automate common system administration tasks, improve security, deploy services, manage containers, and maintain the infrastructure as code.

---

## Project Overview

This project was built to develop practical experience with:

* Linux system administration
* Remote server management
* Infrastructure automation
* Configuration management
* SSH security
* Firewall management
* Service management
* Web server deployment
* Bash automation
* Containerization
* Git version control
* Ansible roles and reusable automation

Rather than manually configuring each server, the environment uses **Ansible as the central automation layer**.

The final environment consists of:

* 1 Ansible control node
* 2 Rocky Linux managed servers
* Host-only networking
* SSH key-based authentication
* Automated system configuration
* Hardened SSH configuration
* Firewalld configuration
* Apache web server deployment
* Podman container deployment
* Bash-based system health reporting
* Git/GitHub version control

---

# Architecture

```text
                         Windows Host
                              │
                    VirtualBox Host-Only
                         192.168.56.0/24
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
        ┌───────────┐   ┌───────────┐   ┌───────────┐
        │ ansible01 │   │  server01 │   │  server02 │
        │           │   │           │   │           │
        │ Control   │   │ Managed   │   │ Managed   │
        │ Node      │   │ Node      │   │ Node      │
        │           │   │           │   │           │
        │ .20       │   │ .10       │   │ .11       │
        └─────┬─────┘   └───────────┘   └───────────┘
              │
              │ Ansible / SSH
              │
       ┌──────┴────────┐
       │               │
       ▼               ▼
    server1         server2
       │               │
    Podman           Podman
       │               │
 Apache Container   Apache Container
   :8080 → :80       :8080 → :80
```

## Network Configuration

| Host         | Role                 | IP Address      |
| ------------ | -------------------- | --------------- |
| `ansible01`  | Ansible Control Node | `192.168.56.20` |
| `server1`    | Managed Server       | `192.168.56.10` |
| `server2`    | Managed Server       | `192.168.56.11` |
| Windows Host | VirtualBox Host      | `192.168.56.1`  |

Network type:

```text
VirtualBox Host-Only Adapter
```

Subnet:

```text
192.168.56.0/24
```

The host-only network allows the virtual machines to communicate with each other and the Windows host without requiring the servers to be directly exposed to the external network.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/d98684e3-1c7a-4ec3-a22c-68f06a12e954" />

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/ef77d8d6-9e55-4724-af25-a0b2916930ee" />

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/c00491b2-777d-4e8a-b055-311a981d52e3" />


---

# Technologies Used

| Technology         | Purpose                   |
| ------------------ | ------------------------- |
| Rocky Linux 9.8    | Linux operating system    |
| VirtualBox 7.2.16  | Virtualization            |
| Ansible Core       | Infrastructure automation |
| SSH                | Remote administration     |
| Firewalld          | Host firewall             |
| Apache HTTP Server | Web service               |
| Podman 5.8.2       | Containerization          |
| Bash               | System automation         |
| Git 2.52.0         | Version control           |
| GitHub             | Remote repository         |

---

# Project Structure

The project is organized as follows:

```text
ansible-lab/
│
├── README.md
├── ansible.cfg
├── inventory
│
├── playbooks/
│   ├── base_setup.yml
│   ├── create_users.yml
│   ├── firewall.yml
│   ├── ssh_hardening.yml
│   ├── web_service.yml
│   └── podman.yml
│
└── scripts/
    └── system_health.sh
```

Ansible roles will be added as the final implementation stage.

---

# 1. Ansible Control Node

`ansible01` acts as the central control node.

Ansible was installed using Rocky Linux's package management system.

```bash
sudo dnf update -y
sudo dnf install epel-release -y
sudo dnf install ansible-core -y
```

Installation was verified with:

```bash
ansible --version
```

The control node manages both Rocky Linux servers remotely through SSH.

---

# 2. Inventory

The Ansible inventory defines the managed servers.

Example:

```ini
[servers]
server1 ansible_host=192.168.56.10
server2 ansible_host=192.168.56.11
```

The `servers` group allows the same automation to be applied to both machines.

For example:

```bash
ansible servers -m ping
```

Expected result:

```text
server1 | SUCCESS
server2 | SUCCESS
```

This confirmed that Ansible could communicate with both managed nodes.

<img width="578" height="308" alt="image" src="https://github.com/user-attachments/assets/6921076a-d037-43f6-84f4-477aa90e59af" />


---

# 3. SSH Configuration

SSH is used by Ansible to communicate with the managed servers.

Key-based authentication was configured so that the control node could connect to the managed nodes without repeatedly entering an SSH password.

The environment therefore follows:

```text
ansible01
   │
   ├── SSH key ──→ server1
   │
   └── SSH key ──→ server2
```

This provides the foundation for automated remote administration.

---

# 4. Base System Configuration

The `base_setup.yml` playbook establishes a basic software and time-synchronization configuration on the managed servers.

Packages configured include:

* Vim
* Git
* Curl
* Wget
* Net-tools

The playbook also ensures that `chronyd` is running and enabled.

Example Ansible configuration:

```yaml
- name: Install base packages
  ansible.builtin.dnf:
    name:
      - vim
      - git
      - curl
      - wget
      - net-tools
    state: present
```

Service management:

```yaml
- name: Ensure chronyd is running
  ansible.builtin.systemd_service:
    name: chronyd
    state: started
    enabled: true
```

This establishes a consistent baseline across both managed nodes.

---

# 5. Automated User Management

The `create_users.yml` playbook was used to automate creation of a user account across the managed servers.

The configuration demonstrated:

* User creation
* Group assignment
* Supplementary groups
* Administrative access
* Idempotent configuration

The `wheel` group was used to provide administrative privileges.

An important Ansible concept demonstrated here was **idempotency**.

Running the same playbook multiple times does not continually create duplicate users or modify the system unnecessarily.

---

# 6. SSH Hardening

SSH was hardened using Ansible.

The initial configuration was inspected before making changes.

```bash
ansible servers -b -m command -a "/usr/sbin/sshd -T" --ask-become-pass
```

The initial effective configuration included:

```text
permitrootlogin yes
passwordauthentication yes
pubkeyauthentication yes
```

The desired configuration was:

```text
permitrootlogin no
passwordauthentication no
pubkeyauthentication yes
```

The `ssh_hardening.yml` playbook uses `lineinfile` to enforce the desired configuration.

Configuration validation was performed before applying changes:

```yaml
validate: "/usr/sbin/sshd -t -f %s"
```

The SSH service was then reloaded so the new configuration could take effect.

## Configuration Override Troubleshooting

During implementation, changing the main `/etc/ssh/sshd_config` file did not completely change the effective configuration.

Investigation revealed an additional configuration file:

```text
/etc/ssh/sshd_config.d/01-permitrootlogin.conf
```

which contained:

```text
PermitRootLogin yes
```

This drop-in configuration was overriding the expected setting.

The automation was updated to handle this additional configuration source.

Final verification showed:

```text
permitrootlogin no
passwordauthentication no
pubkeyauthentication yes
```

This demonstrated the importance of checking the **effective configuration** rather than assuming that modifying the main configuration file is sufficient.

<img width="803" height="692" alt="Screenshot 2026-09-13 232411" src="https://github.com/user-attachments/assets/2c3249a2-d109-4427-842f-f6f58797ce2b" />

<img width="795" height="622" alt="Screenshot 2026-09-13 232433" src="https://github.com/user-attachments/assets/6357a396-6ead-43c9-b754-9bf1ea6a105d" />

<img width="809" height="655" alt="Screenshot 2026-09-13 232455" src="https://github.com/user-attachments/assets/270a0a24-39f5-4c4e-a732-30ae0a4c3d2d" />

---

# 7. Firewalld

Firewalld was configured using Ansible.

The `firewall.yml` playbook ensures that Firewalld is:

* Installed
* Running
* Enabled at boot
* Configured to allow SSH
* Configured to allow HTTP

The `ansible.posix.firewalld` module was used.

The required Ansible collection was installed with:

```bash
ansible-galaxy collection install ansible.posix
```

Example:

```yaml
- name: Allow HTTP
  ansible.posix.firewalld:
    service: http
    permanent: true
    immediate: true
    state: enabled
    zone: public
```

Two important Firewalld concepts were demonstrated:

### Permanent

```text
permanent: true
```

ensures the configuration survives a reload/reboot.

### Immediate

```text
immediate: true
```

applies the rule to the current runtime configuration.

Using only `permanent: true` does not necessarily make the rule immediately active in the current runtime configuration.

<img width="804" height="613" alt="Screenshot 2026-09-14 181228" src="https://github.com/user-attachments/assets/ce711dac-bcd6-4f1a-b6fb-110cc8a25f60" />

<img width="799" height="119" alt="Screenshot 2026-09-14 181307" src="https://github.com/user-attachments/assets/98f0d762-3a64-4163-b374-b8ef58baae88" />

<img width="784" height="567" alt="Screenshot 2026-09-14 181341" src="https://github.com/user-attachments/assets/638411e4-40cb-4aff-b481-88e8ccba21bc" />

<img width="913" height="820" alt="Screenshot 2026-09-14 181521" src="https://github.com/user-attachments/assets/27ab601d-30c6-4e1a-a72f-a6c43ae81a7a" />

<img width="769" height="500" alt="Screenshot 2026-09-14 181538" src="https://github.com/user-attachments/assets/efcff5ba-6d2c-4c71-99c2-d276f375d5ec" />

## 🔐 Node Isolation & Management

As part of the infrastructure hardening process, additional access controls were applied to the managed nodes to reduce unnecessary exposed services and standardize node configuration.

### Access Management

The firewall configuration was updated across the managed Rocky Linux servers to:

* Keep SSH available for Ansible-based remote administration
* Allow HTTP traffic on port 80
* Allow TCP port 8080 for the Podman-hosted Apache service
* Remove Cockpit from the enabled firewall services
* Apply changes both immediately and permanently

The resulting firewall configuration was verified with:

```bash
sudo firewall-cmd --list-all
```

Example:

```text
services: dhcpv6-client http ssh
ports: 8080/tcp
```

### Centralized Configuration

Firewall changes were applied through Ansible rather than manually configuring each node. This allows the same access configuration to be consistently deployed across the managed infrastructure.

```yaml
- name: Disabling cockpit
  ansible.posix.firewalld:
    state: disabled
    service: cockpit
    immediate: true
    permanent: true

- name: Allow port 8080 TCP for Podman
  ansible.posix.firewalld:
    state: enabled
    port: 8080/tcp
    permanent: true
    immediate: true
```

This provides a foundation for further node isolation and network access policies as the infrastructure grows.

<img width="929" height="619" alt="Screenshot 2026-09-17 122749" src="https://github.com/user-attachments/assets/e7253d77-770f-49e7-af7b-6406d3aa9ae6" />


---

# 8. Service Management

Service management was incorporated throughout the project rather than being treated as a separate isolated component.

Ansible's `systemd_service` module was used to manage services such as:

* `chronyd`
* `httpd`

Example:

```yaml
- name: Ensure Apache is running
  ansible.builtin.systemd_service:
    name: httpd
    state: started
    enabled: true
```

The project therefore demonstrates both:

```text
state: started
```

and:

```text
enabled: true
```

The first ensures that the service is running now, while the second ensures it starts automatically during boot.

---

# 9. Apache Web Server Deployment

Apache HTTP Server was deployed to both managed servers using Ansible.

The `web_service.yml` playbook performs the following:

1. Installs Apache
2. Ensures Apache is running
3. Enables Apache at boot
4. Deploys a custom web page

Example installation:

```yaml
- name: Install Apache
  ansible.builtin.dnf:
    name: httpd
    state: present
```

Service management:

```yaml
- name: Start and enable Apache
  ansible.builtin.systemd_service:
    name: httpd
    state: started
    enabled: true
```

## Port Conflict Troubleshooting

One managed server already had Nginx installed from an earlier stage of the lab.

Nginx was occupying port 80, which prevented Apache from binding to the same port.

Apache reported:

```text
AH00072: make_sock: could not bind to address [::]:80
```

The conflicting Nginx package was removed through Ansible:

```yaml
- name: Removing Port 80 conflicts
  ansible.builtin.dnf:
    name: nginx
    state: absent
    autoremove: true
```

Apache could then start normally.

This demonstrated practical troubleshooting of a service startup failure caused by a port conflict.

<img width="1381" height="930" alt="Screenshot 2026-09-14 190951" src="https://github.com/user-attachments/assets/fee632a9-dfe1-4317-8c82-2cbcc1bcbcfa" />

<img width="966" height="210" alt="Screenshot 2026-09-14 191255" src="https://github.com/user-attachments/assets/adcd2178-df7b-42a6-9cbb-810217c83731" />

---

# 10. Custom Web Page

A custom HTML page was deployed using Ansible's `copy` module.

Example:

```yaml
- name: Deploy custom webpage
  ansible.builtin.copy:
    dest: /var/www/html/index.html
    content: |
      <h1>RHEL Automation Lab</h1>
      <p>Managed by Ansible.</p>
```

The deployment was verified remotely using:

```bash
curl -I 192.168.56.10
curl -I 192.168.56.11
```

The initial response was:

```text
HTTP/1.1 403 Forbidden
```

which still confirmed that the request was reaching Apache.

After deploying the correct index page, the response became:

```text
HTTP/1.1 200 OK
```

This verified the complete web-service path:

```text
ansible01
   │
   │ HTTP
   ▼
server1/server2
   │
 Apache
   │
index.html
```
<img width="575" height="328" alt="Screenshot 2026-09-14 191835" src="https://github.com/user-attachments/assets/1d6aa85d-5cfa-496a-9c1d-3f7d9b01346d" />


<img width="751" height="850" alt="Screenshot 2026-09-14 192838" src="https://github.com/user-attachments/assets/8f00eaa1-3b27-4894-a638-987bc85cab91" />

---

# 11. Bash System Health Automation

A Bash script was created to automate system health checks across the managed servers.

The script uses Ansible to retrieve the managed host list:

```bash
ansible servers --list-hosts
```

The host list is processed using command substitution and standard Unix text-processing tools.

The script then loops through the servers and executes a consolidated health-check command.

The health report collects:

* Hostname
* Disk usage
* Memory usage
* System uptime
* Running services
* Logged-in users

Commands used include:

```bash
hostname
df -h
free -h
uptime
systemctl list-units --type=service --state=running
w
```

Instead of launching a separate Ansible command for every individual check, the script was optimized to send the checks as one remote shell command per server.

Conceptually:

```text
Bash Script
    │
    ├── server1
    │     └── one Ansible call
    │           ├── hostname
    │           ├── disk
    │           ├── memory
    │           ├── uptime
    │           ├── services
    │           └── users
    │
    └── server2
          └── one Ansible call
                ├── hostname
                ├── disk
                ├── memory
                ├── uptime
                ├── services
                └── users
```

This combines Bash scripting with Ansible automation.

---

# 12. Podman Containerization

Podman was already available on both managed servers.

Version:

```text
Podman 5.8.2
```

The Ansible `containers.podman` collection was used to manage containers.

The module used was:

```text
containers.podman.podman_container
```

A containerized Apache HTTP server was deployed to both managed nodes.

The desired configuration was:

```text
Host Port 8080
       │
       ▼
Container Port 80
       │
       ▼
Apache HTTP Server
```

The container configuration uses:

```yaml
containers.podman.podman_container:
  name: apache-container
  image: docker.io/library/httpd:latest
  state: started
  publish: "8080:80"
```

This results in:

```text
server1:8080 → apache-container:80
server2:8080 → apache-container:80
```

## Container Verification

Containers were inspected remotely using:

```bash
ansible servers -b -m shell -a "podman ps -a" --ask-become-pass
```

Both servers reported:

```text
apache-container
```

with:

```text
0.0.0.0:8080->80/tcp
```

The application was then tested from the Ansible control node:

```bash
curl http://192.168.56.10:8080
curl http://192.168.56.11:8080
```

Both returned the Apache response successfully.

---

# 13. Rootful vs Rootless Podman

An important troubleshooting issue occurred during container verification.

The Ansible playbook used:

```yaml
become: true
```

which caused the container to be created in the **rootful Podman context**.

Running:

```bash
podman ps
```

as `ansibleadmin` returned no containers.

However:

```bash
ansible servers -b -m shell -a "podman ps -a" --ask-become-pass
```

showed the running containers.

This demonstrated that rootful and rootless Podman use different container storage and execution contexts.

The issue was therefore not that the containers had failed to deploy; they were being inspected from a different user context.

---

# 14. Git Version Control

Git was already installed:

```text
Git 2.52.0
```

The project was initialized as a Git repository:

```bash
git init
```

Project files were staged:

```bash
git add .
```

The initial commit was created with:

```bash
git commit -m "Initial infrastructure lab setup"
```

The repository was then connected to GitHub using an SSH remote.

Example:

```text
git@github.com:USERNAME/REPOSITORY.git
```

---

# 15. GitHub SSH Authentication

SSH key authentication was configured for GitHub.

The existing Ed25519 public key was added to GitHub.

The private key remains on the Linux control node and is not stored in the repository.

## SSH Port 22 Issue

The first GitHub SSH connection attempted to use the standard SSH port:

```text
22
```

The connection failed with:

```text
ssh: connect to host github.com port 22: Connection refused
```

GitHub's SSH service over port 443 was then used:

```bash
ssh -T -p 443 git@ssh.github.com
```

Authentication succeeded.

The SSH client configuration was updated so GitHub connections automatically use port 443:

```text
Host github.com
    HostName ssh.github.com
    Port 443
    User git
```

The normal connection was then tested:

```bash
ssh -T git@github.com
```

Successful authentication confirmed that the GitHub SSH key and SSH configuration were working.

The repository was subsequently pushed to GitHub.

---

# 16. Ansible Roles

**Status: Pending final implementation**

The final implementation stage is to reorganize selected automation into reusable Ansible roles.

The purpose of this stage is to demonstrate:

* Reusable automation
* Role structure
* Separation of configuration responsibilities
* Variables
* Defaults
* Handlers
* Tasks
* Templates/files where appropriate

Planned role structure:

```text
roles/
├── base/
├── ssh_hardening/
├── firewall/
├── web_service/
└── podman/
```

The exact roles and tasks will be finalized after implementation.

---

# 17. Idempotency

A major design principle throughout the project is **idempotency**.

Ansible tasks are designed to describe the desired state rather than simply execute commands repeatedly.

For example:

```yaml
state: present
```

means the package should exist.

```yaml
state: started
```

means the service should be running.

```yaml
enabled: true
```

means the service should start automatically.

Running the same playbook multiple times should therefore produce fewer changes after the system has reached the desired state.

Example:

```text
First run:
changed=...

Second run:
changed=0
```

This behavior is one of the main advantages of configuration management compared with manually executing commands on every server.

---

# 18 🎭 Ansible Roles

The base system configuration was refactored from a standalone playbook into a reusable Ansible Role. This separates the configuration logic from the playbook and provides a more maintainable structure for expanding the automation.

### Role Structure

The `base_setup` role was created using `ansible-galaxy init`:

```bash
ansible-galaxy init roles/base_setup
```

The resulting structure is:

```text
roles/
└── base_setup/
    ├── defaults/
    ├── handlers/
    ├── meta/
    ├── tasks/
    │   └── main.yml
    ├── tests/
    └── vars/
```

**📸 Screenshot — Role Structure**

> Add screenshot here showing the `roles/base_setup` directory and `tasks/main.yml`.

### Base Setup Role

The role automates the initial configuration of the managed Rocky Linux servers.

The `tasks/main.yml` file handles:

* Installation of required packages:

  * Vim
  * Git
  * cURL
  * Wget
  * net-tools
* Starting and enabling `chronyd`

Example:

```yaml
- name: Installing required packages
  ansible.builtin.dnf:
    name: [vim, git, curl, wget, net-tools]
    state: present

- name: Start and Enable chronyd
  ansible.builtin.systemd_service:
    name: chronyd
    state: started
    enabled: true
```

### Using the Role

The role is called from a dedicated playbook rather than directly including its task file:

```yaml
- name: Basic Role Setup
  hosts: servers
  become: true

  roles:
    - base_setup
```

Ansible automatically loads the role's `tasks/main.yml` when `base_setup` is specified under `roles`.

### Idempotency

The role was executed multiple times to verify idempotent behavior.

During the initial execution, the required packages were installed and Ansible reported changes.

A subsequent execution produced:

```text
server1 : ok=3 changed=0 unreachable=0 failed=0
server2 : ok=3 changed=0 unreachable=0 failed=0
```

This confirms that once the servers reached the desired state, running the role again did not make unnecessary changes.

<img width="957" height="386" alt="Screenshot 2026-09-17 123005" src="https://github.com/user-attachments/assets/f9455356-bf42-4fa4-b354-5f0852c8907b" />


> Add screenshot here showing the successful second execution with `changed=0` and `failed=0`.

### Verification

The role was validated using:

```bash
ansible-playbook --syntax-check playbooks/base_role.yml
```

The playbook then executed successfully against both managed nodes.

This demonstrates the use of reusable roles, privilege escalation, package management, systemd service management, and idempotent configuration management.


# 19. Troubleshooting Experience

Several real configuration and infrastructure issues were encountered during development.

## SSH Configuration Override

### Problem

Changing `/etc/ssh/sshd_config` did not change the effective `PermitRootLogin` setting.

### Investigation

The effective configuration was inspected using:

```bash
sshd -T
```

An additional configuration file was discovered:

```text
/etc/ssh/sshd_config.d/01-permitrootlogin.conf
```

### Solution

The Ansible automation was updated to account for the drop-in configuration.

---

## Firewalld Runtime vs Permanent Configuration

### Problem

HTTP was configured permanently but was not immediately active in the runtime configuration.

### Solution

The task was updated to use:

```yaml
permanent: true
immediate: true
```

This ensured the firewall rule was both persistent and immediately applied.

---

## Apache Port Conflict

### Problem

Apache failed to start because port 80 was already occupied.

### Error

```text
AH00072: make_sock: could not bind to address [::]:80
```

### Cause

Nginx was already running on the affected server.

### Solution

Nginx was removed through Ansible before starting Apache.

---

## Podman Container Visibility

### Problem

`podman ps` showed no containers after Ansible reported that the container had changed.

### Investigation

The playbook used:

```yaml
become: true
```

while the verification command did not.

### Solution

The container was inspected using root privileges:

```bash
ansible servers -b -m shell -a "podman ps -a" --ask-become-pass
```

The containers were found running successfully.

---

## GitHub SSH Port

### Problem

SSH connections to GitHub on port 22 were refused.

### Solution

GitHub's SSH endpoint over port 443 was configured.

This allowed Git operations to use SSH authentication without relying on port 22.

### Ansible Role Not Found

**Problem:**
After creating the `base_setup` role, Ansible could not find it when running the role-based playbook:

```text
ERROR! the role 'base_setup' was not found
```

The role existed under the project's `roles/` directory, but that directory was not included in Ansible's role search path.

**Solution:**
Updated `ansible.cfg` to explicitly define the project role path:

```ini
[defaults]
inventory = inventory
roles_path = ./roles
```

The playbook was then able to locate and execute the `base_setup` role successfully.

---

### Firewall Configuration Interrupted Ansible Connectivity

**Problem:**
While experimenting with more restrictive SSH firewall rules, the managed nodes became unreachable from the Ansible control node because SSH access was no longer permitted.

**Solution:**
Access to the VM consoles was used to restore the SSH firewall service:

```bash
sudo firewall-cmd --zone=public --add-service=ssh
```

Ansible connectivity was then restored.

The final firewall configuration was kept focused on the services required by the project:

* SSH for remote administration
* HTTP for Apache
* TCP port 8080 for the Podman service
* Cockpit removed

**Lesson:**
Firewall changes can directly affect the remote management channel. Changes to access-control rules should be tested carefully, with console access available for recovery.

---

### Quadlet Service Not Appearing in Systemd

**Problem:**
After creating `/etc/containers/systemd/apache.container`, the expected systemd service was not immediately visible. The Quadlet configuration existed, but systemd had not yet regenerated its unit configuration.

**Solution:**
Reloaded the systemd configuration:

```bash
sudo systemctl daemon-reload
```

The generated service then appeared as:

```text
apache.service
```

The service was verified with:

```bash
systemctl status apache.service
```

The Ansible playbook was also configured to automatically reload systemd when deploying the Quadlet configuration:

```yaml
- name: Reload systemd and start Apache container
  ansible.builtin.systemd_service:
    daemon_reload: true
    name: apache.service
    enabled: true
    state: started
```

The container was subsequently verified to start automatically after reboot.


---

# 20. Verification Strategy

The infrastructure was verified at multiple levels rather than relying solely on Ansible's task output.

### Ansible connectivity

```bash
ansible servers -m ping
```

### SSH configuration

```bash
ansible servers -b -m command -a "/usr/sbin/sshd -T" --ask-become-pass
```

### Service status

```bash
ansible servers -b -m shell -a "systemctl is-active httpd" --ask-become-pass
```

### Podman containers

```bash
ansible servers -b -m shell -a "podman ps -a" --ask-become-pass
```

### Web server

```bash
curl http://192.168.56.10
curl http://192.168.56.11
```

### Containerized web server

```bash
curl http://192.168.56.10:8080
curl http://192.168.56.11:8080
```

### Git repository

```bash
git status
git log --oneline
git remote -v
```

### GitHub SSH

```bash
ssh -T git@github.com
```

---

# 21. Key Concepts Demonstrated

This project provided hands-on experience with:

### Linux Administration

* Package management with DNF
* Users and groups
* Administrative privileges
* Services
* SSH
* Filesystem configuration
* Network interfaces
* Firewall configuration
* System monitoring

### Ansible

* Inventory management
* Ad-hoc commands
* Playbooks
* Modules
* Variables
* Privilege escalation
* Idempotency
* Collections
* Service management
* Configuration validation
* Container management
* Roles

### Bash

* Variables
* Command substitution
* Loops
* Remote command execution
* Pipes
* Text processing
* System monitoring
* Script-based orchestration

### Containers

* Podman
* Container images
* Container lifecycle
* Port publishing
* Rootful containers
* Containerized web services

### Networking

* Host-only networking
* IP addressing
* SSH
* TCP ports
* Port conflicts
* HTTP
* Firewall rules
* Port forwarding/publishing

### Git

* Repository initialization
* Staging
* Commits
* Branches
* Remotes
* SSH authentication
* GitHub integration

---

# 22. Screenshots

The following screenshots document important stages of the implementation.

### Ansible Connectivity

<img width="578" height="308" alt="Screenshot 2026-09-15 213147" src="https://github.com/user-attachments/assets/2c37e252-0400-443a-912b-ec8db92bd105" />


```text
ansible servers -m ping
```

### SSH Hardening

<img width="951" height="208" alt="image" src="https://github.com/user-attachments/assets/4ae22c64-3543-4771-b4a2-cdfa9d347246" />


```text
sshd -T
```

showing:

```text
permitrootlogin no
passwordauthentication no
pubkeyauthentication yes
```

### Firewalld

<img width="929" height="619" alt="Screenshot 2026-09-17 122749" src="https://github.com/user-attachments/assets/fe7ca1ca-10cb-4dab-b823-80e02fbf8598" />


Show HTTP/SSH services configured in the active firewall zone.


Show:

```text
systemctl status httpd
```

or the successful HTTP response.

### Custom Webpage

<img width="751" height="850" alt="Screenshot 2026-09-14 192838" src="https://github.com/user-attachments/assets/1fadd59a-c8be-45ce-82b8-eb2168d35652" />


Show the custom webpage being served.

### Bash Health Report

<img width="957" height="528" alt="Screenshot 2026-09-14 203716" src="https://github.com/user-attachments/assets/8ca17f69-0799-454c-9725-a5eba0959227" />


<img width="1418" height="900" alt="Screenshot 2026-09-14 202925" src="https://github.com/user-attachments/assets/807c92a4-52d3-4a3e-b240-1904f2b4e11f" />

<img width="1418" height="943" alt="Screenshot 2026-09-14 202945" src="https://github.com/user-attachments/assets/b5a76ea5-d533-4b40-917c-4d88e0b6e2bd" />

Show the system health output for both servers.

### Podman

<img width="940" height="844" alt="Screenshot 2026-09-15 203926" src="https://github.com/user-attachments/assets/bb3f364d-6365-45c8-b175-c12b869f8775" />

<img width="805" height="390" alt="Screenshot 2026-09-15 203940" src="https://github.com/user-attachments/assets/4b510292-c7df-4712-ab71-36ac855675ec" />

Show:

```text
podman ps -a
```

with:

```text
apache-container
0.0.0.0:8080->80/tcp
```

### Containerized Apache

<img width="805" height="390" alt="Screenshot 2026-09-15 203940" src="https://github.com/user-attachments/assets/687ee0ec-54c5-42da-bf97-339c102dbb6b" />


Show:

```bash
curl http://192.168.56.10:8080
```

and/or:

```bash
curl http://192.168.56.11:8080
```

### Git

<!-- ADD SCREENSHOT -->

Show:

```bash
git log --oneline
```

and/or the GitHub repository containing the project.

---

# 22. Future Improvements

Possible future improvements include:

* Complete Ansible role conversion
* Add Ansible handlers
* Introduce variables for configurable ports and packages
* Add Jinja2 templates
* Add automated testing
* Add container health checks
* Add centralized logging
* Add monitoring
* Add automated backup configuration
* Add CI validation for Ansible playbooks
* Add Ansible Vault for sensitive variables
* Expand the environment with additional managed nodes
* Add a reverse proxy
* Add a database container
* Add automated deployment pipelines

---

# 23. Project Outcome

This project evolved from a basic Rocky Linux virtual-machine environment into a multi-node infrastructure automation lab.

The final architecture demonstrates the ability to:

```text
Configure Linux
      ↓
Manage users
      ↓
Secure SSH
      ↓
Configure firewall
      ↓
Manage services
      ↓
Deploy web services
      ↓
Automate system health checks
      ↓
Deploy containers
      ↓
Version-control infrastructure
      ↓
Automate through reusable Ansible roles
```

The project emphasizes **automation, repeatability, troubleshooting, and infrastructure-as-code principles** rather than simply installing individual technologies.

---

# Author

**Prerak Patel**

Computer Science,
University of Lethbridge

GitHub: `PrerakP13`

---

# License

This project is intended for educational, portfolio, and demonstration purposes.
