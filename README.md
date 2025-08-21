# Ansible + Terraform: EC2 Web Server

This project provisions an AWS EC2 instance with Terraform and configures it as a basic web server using an Ansible role.

### What it does

- Provisions networking (VPC, public subnet, internet gateway, route table) in `eu-central-1`.
- Creates an EC2 instance (Ubuntu, `t2.micro`) with a public IP and an SSH security group.
- Installs and starts Nginx, creates an `appuser`, and installs basic tools via an Ansible role `webserver`.

## Prerequisites

- AWS account and credentials configured (e.g., via `aws configure`).
- Terraform installed.
- Ansible installed.
- An SSH key pair available locally.

## Repository layout

- `main.tf`: Terraform for VPC, subnet, route table, security group, key pair, AMI lookup, and EC2 instance.
- `site.yml`: Ansible play applying the `webserver` role to `webservers` group.
- `ansible.cfg`: Points to role path and default inventory; configures SSH private key and disables host key checking for convenience.
- `ansible-webserver/webserver`: Ansible role containing tasks, variables, handlers, and tests.

## Important defaults and hardcoded values

- Region: `eu-central-1` and AZ `eu-central-1a`.
- SSH ingress is restricted to a single IP: `176.100.28.228/32` (update to your IP).
- Terraform key pair name: `my-key` with public key path `~/.ssh/my-key.pub`.
- Ansible uses private key `~/.ssh/my-key` via `ansible.cfg`.
- Default Ansible inventory: `ansible-webserver/webserver/tests/inventory`.

Update these to match your environment before running.

## Setup and usage

### 1) Prepare SSH key

Ensure you have an SSH key pair and that the public key path matches `main.tf`:

```bash
ls -l ~/.ssh/my-key ~/.ssh/my-key.pub
```

If needed, create one:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/my-key -N ""
```

### 2) Provision infrastructure with Terraform

```bash
terraform init
terraform apply -auto-approve
```

On completion, Terraform will output the instance public IP as `instance_ip`.

To query it again later:

```bash
terraform output -raw instance_ip
```

### 3) Update Ansible inventory

By default, the project uses `ansible-webserver/webserver/tests/inventory`. Replace the placeholder IP with your instance IP and ensure user and key are correct:

```ini
[webservers]
<INSTANCE_PUBLIC_IP> ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/my-key
```

Note: Ubuntu AMIs typically use the `ubuntu` SSH user.

### 4) Test connectivity

```bash
ansible -i ansible-webserver/webserver/tests/inventory webservers -m ping
```

### 5) Configure the server with Ansible

```bash
ansible-playbook site.yml
```

This will:

- Update and upgrade packages
- Install `git`, `curl`, `htop`
- Create the `appuser`
- Install, start, and enable `nginx`

## Security notes

- Restrict SSH access: update the security group ingress in `main.tf` to your current IP or a controlled CIDR.
- Manage keys securely: avoid committing private keys; rotate keys regularly.
- Consider enabling host key checking in `ansible.cfg` for stricter security in non-demo environments.

## Cleanup

```bash
terraform destroy -auto-approve
```

## Troubleshooting

- Permission denied (publickey): verify the Key Pair name in Terraform, the uploaded public key, and the inventory `ansible_ssh_private_key_file` path.
- Timeout connecting: ensure the instance has a public IP, the route table is associated, and your client IP matches the SSH security group rule.
- Ansible host key warnings: either accept the host key manually or keep `host_key_checking=False` (default here).

## Role details (summary)

Role `webserver` tasks:

- Update/upgrade apt cache
- Install `git`, `curl`, `htop`
- Create `appuser`
- Install and enable `nginx`
