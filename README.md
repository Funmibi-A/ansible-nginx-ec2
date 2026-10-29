# Deploying Nginx on AWS EC2 with Ansible

Automated deployment of an Nginx web server on an Amazon EC2 instance using Ansible. An EC2 instance is created in AWS, secured with a Security Group, and then configured entirely from my local machine with an Ansible playbook that installs Nginx and serves a custom HTML page.

## Overview

This project separates **infrastructure provisioning** (AWS EC2 and networking) from **configuration management** (Ansible). Once the server exists, Ansible connects to it over SSH and brings it to a defined state:

- System packages updated
- Nginx installed
- Nginx started and enabled at boot
- A custom HTML page deployed to the web root

Because Ansible describes the *desired state* rather than a list of commands, the playbook is **idempotent**: running it again changes nothing if the server already matches.

## Architecture
![network diagram](screenshot/ec2-nginx-ansible-network-diagram.png)

| Component | Details |
|---|---|
| Cloud provider | AWS |
| Compute | EC2 `t2.micro` (free tier) |
| OS | Ubuntu Server 24.04 LTS |
| Control node | Local machine running Ansible (WSL) |
| Web server | Nginx 1.24.0 |
| Authentication | SSH key pair (`.pem`) |

## Project Structure

```
ansible-nginx-ec2/
├── playbook.yml            # Ansible playbook
├── inventory.ini           # Ansible inventory (EC2 public IP)
├── index.html              # Custom page deployed to the server
├── ansible-recap-1.png     # Screenshot: first playbook run
├── ansible-recap-2.png     # Screenshot: second playbook run (idempotency)
├── nginx.png               # Screenshot: deployed page in the browser
├── .gitignore
└── README.md
```

## Prerequisites

- An AWS account
- Ansible installed locally (`ansible [core 2.16.3]` used here)
- Git and a GitHub account
- An SSH key pair created in AWS and saved locally with restricted permissions:

```bash
chmod 400 ~/.ssh/ansible-nginx-key.pem
```

## Infrastructure Setup

### 1. SSH key pair

Created in **EC2 → Network & Security → Key Pairs** as `ansible-nginx-key` (`.pem` format). The private key is stored in `~/.ssh/` and is **never committed to Git**.

### 2. Security Group (`nginx-web-sg`)

| Type | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| SSH | TCP | 22 | My IP | Admin access, restricted to my own address |
| HTTP | TCP | 80 | 0.0.0.0/0 | Public web traffic |

### 3. EC2 instance

- Name: `ansible-nginx-server`
- AMI: Ubuntu Server 24.04 LTS
- Instance type: `t2.micro` / `t3.micro`
- Key pair: `ansible-nginx-key`
- Security group: `nginx-web-sg`
- Public IPv4 auto-assign: enabled

### 4. Manual SSH test

Before using Ansible, I confirmed plain SSH worked, since Ansible relies on the same connection:

```bash
ssh -i ~/.ssh/ansible-nginx-key.pem ubuntu@54.210.37.254
```

## Ansible Configuration

### Inventory (`inventory.ini`)

```ini
[webservers]
nginx_server ansible_host=54.210.37.254

[webservers:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/ansible-nginx-key.pem
ansible_python_interpreter=/usr/bin/python3
```

Connectivity test:

```bash
ansible -i inventory.ini webservers -m ping
```

```
nginx_server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

> The Ansible `ping` module is not an ICMP ping. It logs in over SSH and runs a small Python script, so it only needs port 22.

### Playbook (`playbook.yml`)

```yaml
---
- name: Configure Nginx web server on EC2
  hosts: webservers
  become: true

  tasks:
    - name: Update apt package cache and upgrade system packages
      ansible.builtin.apt:
        update_cache: true
        upgrade: dist
        cache_valid_time: 3600

    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: Start and enable Nginx service
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

    - name: Deploy custom HTML page
      ansible.builtin.copy:
        src: index.html
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: "0644"
      notify: Reload Nginx

  handlers:
    - name: Reload Nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

| Task | What it does |
|---|---|
| Update apt cache and upgrade | Refreshes package lists and upgrades installed packages |
| Install Nginx | Ensures the `nginx` package is present |
| Start and enable Nginx | Runs the service now and on every reboot |
| Deploy custom HTML | Copies `index.html` to the web root with correct ownership and permissions |
| Handler: Reload Nginx | Runs only when the HTML file actually changed |

## Running the Playbook

Check syntax first, then run:

```bash
ansible-playbook -i inventory.ini playbook.yml --syntax-check
ansible-playbook -i inventory.ini playbook.yml
```

## Idempotency Demonstration

I ran the same playbook twice.

| | Run 1 | Run 2 |
|---|---|---|
| ok | 6 | 5 |
| changed | **4** | **0** |
| failed | 0 | 0 |

**Run 1** applied the apt update, installed Nginx, deployed the HTML file, and triggered the reload handler. The "Start and enable Nginx" task reported `ok` rather than `changed` because Ubuntu already starts and enables Nginx automatically when the package is installed.

![Run 1 recap](ansible-recap-1.png)

**Run 2** reported `changed=0`. Ansible compared the server to the desired state, found nothing to fix, and did nothing. The reload handler did not run because the HTML file was unchanged.

![Run 2 recap](ansible-recap-2.png)

This is idempotency: the playbook can be run repeatedly and safely, which helps prevent configuration drift.

## Validation

Opening `http://54.210.37.254` in a browser shows the custom page served by Nginx:

![Deployed page](nginx.png)

Command-line check:

```bash
curl -I http://54.210.37.254
```

Expected: `HTTP/1.1 200 OK` with `Server: nginx/1.24.0 (Ubuntu)`.

## Troubleshooting

### 1. Ansible ping seemed to hang

- **Symptom:** `ansible -m ping -vvv` printed the SSH command and then appeared to stop.
- **Investigation:** I tested SSH manually with the same key and user, and it worked. The inventory file was correct.
- **Cause:** The first connection to a new instance can take several seconds. I had not waited long enough for the result.
- **Resolution:** Re-ran the command and waited for the prompt to return. It returned `pong`.
- **Lesson:** Give SSH time, and read the full output before changing configuration. `ConnectTimeout` is 10 seconds by default, so the error appears only after that delay.

### 2. Nginx page did not load (port 80 timed out)

- **Symptom:** SSH and Ansible worked, but `curl -I http://<EC2_PUBLIC_IP>` failed with `Couldn't connect to server` after about 140 seconds. The browser could not load the page either.
- **Investigation:**
  1. Proved Nginx was healthy by checking it from inside the server through Ansible:
     ```bash
     ansible -i inventory.ini webservers -a "curl -sI http://localhost"
     ```
     This returned `HTTP/1.1 200 OK`, so the playbook and Nginx were fine and the problem was the network path.
  2. Queried the EC2 instance metadata service to see which security group was actually attached:
     ```bash
     ansible -i inventory.ini webservers -a 'bash -c "TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H \"X-aws-ec2-metadata-token-ttl-seconds: 60\"); curl -s -H \"X-aws-ec2-metadata-token: $TOKEN\" http://169.254.169.254/latest/meta-data/security-groups"'
     ```
     The output was `launch-wizard-1`, not `nginx-web-sg`.
- **Cause:** The instance was attached to the default security group created by the launch wizard, which only allows SSH. Port 80 was being silently dropped.
- **Resolution:** In **EC2 → Instances → Actions → Security → Change security groups**, removed `launch-wizard-1` and added `nginx-web-sg`. The page loaded immediately.
- **Lesson:** A **timeout** (silently dropped packets) points to a firewall or security group. A **connection refused** would point to the service itself. Separating "is the service healthy?" from "is the network path open?" makes diagnosis much faster.

### 3. Why a normal `ping` to the instance fails

Only ports 22 and 80 are allowed, so ICMP is blocked and `ping <EC2_PUBLIC_IP>` gets no reply. This is expected. The Ansible `ping` module works over SSH and is unaffected.



## Security Notes

- SSH (port 22) is restricted to my own IP, not the whole internet.
- HTTP (port 80) is open publicly because this is a public website.
- The private key has `chmod 400` permissions and is excluded from Git via `.gitignore`.
- Key-based authentication is used, with password authentication disabled in the SSH options Ansible uses.
- Only the minimum required ports are open.


#