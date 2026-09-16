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

<!-- ADD SCREENSHOT: VirtualBox VM/network configuration -->

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

<!-- ADD SCREENSHOT: Ansible ping showing both servers -->

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

<!-- ADD SCREENSHOT: SSH effective configuration -->

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

<!-- ADD SCREENSHOT: firewall-cmd output -->

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

<!-- ADD SCREENSHOT: curl 200 OK response -->

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

# 18. Troubleshooting Experience

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

---

# 19. Verification Strategy

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

# 20. Key Concepts Demonstrated

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

# 21. Screenshots

The following screenshots document important stages of the implementation.

Recommended screenshots:

### Ansible Connectivity

<!-- ADD SCREENSHOT -->

```text
ansible servers -m ping
```

### SSH Hardening

<!-- ADD SCREENSHOT -->

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

<!-- ADD SCREENSHOT -->

Show HTTP/SSH services configured in the active firewall zone.

### Apache

<!-- ADD SCREENSHOT -->

Show:

```text
systemctl status httpd
```

or the successful HTTP response.

### Custom Webpage

<!-- ADD SCREENSHOT -->

Show the custom webpage being served.

### Bash Health Report

<!-- ADD SCREENSHOT -->

Show the system health output for both servers.

### Podman

<!-- ADD SCREENSHOT -->

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

<!-- ADD SCREENSHOT -->

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
