# Ansible RoboShop Roles + Terraform

Role-based Ansible automation for the **RoboShop** microservices stack, with EC2 instances **provisioned by Terraform** and targeted dynamically via the AWS EC2 inventory plugin.

This is the integration layer between two worlds: `../terraform-aws-instance/` provisions the compute, and Ansible roles configure each service — all discovered automatically from AWS tags.

## How It Works

```
  Terraform                    Ansible
  ─────────                    ───────
  provisions EC2  ──────────►  aws_ec2 dynamic inventory
  instances with              plugin filters by tag:Name
  tag:Name=frontend-dev       and groups by tags.component
                                │
                                ▼
                         roboshop.yaml
                         (hosts: {{ component }})
                                │
                                ▼
                         roles/<component>/tasks/main.yaml
                         (composes common role task-files)
```

## Key Difference From `../ansible-roboshop-roles/`

| Aspect | `ansible-roboshop-roles` | `ansible-roboshop-roles-tf` (this) |
|---|---|---|
| Inventory | Static `inventory.ini` with DNS names | **Dynamic** `aws_ec2` plugin from AWS tags |
| EC2 provisioning | Done externally (`../roboshop-ansible`) | **Terraform** (`../terraform-aws-instance/`) |
| Host discovery | Manual entry per service | Automatic from `tag:Component` + `tag:Name` |

## Inventory (dynamic)

`frontend.aws_ec2.yaml`:

```yaml
plugin: amazon.aws.aws_ec2
region:
- us-east-1
hostnames:
- private-ip-address
filters:
  instance-state-name: running
  tag:Name: frontend-dev
compose:
  ansible_host: private_ip_address
keyed_groups:
- separator: ''
  key: tags.component
```

The `keyed_groups` directive means EC2 instances tagged `Component: cart` are automatically added to a `cart` inventory group — which is exactly the group name `roboshop.yaml` targets with `hosts: "{{ component }}"`.

## Quick Start

```bash
# Provision instances with Terraform (see ../terraform-aws-instance)
cd ../terraform-aws-instance && terraform apply

# Then configure any service
cd ../ansible-roboshop-roles-tf
ansible-playbook roboshop.yaml -e component=cart
```

## Prerequisites

- AWS credentials for the `amazon.aws` collections (aws_ec2 plugin, SSM lookup)
- EC2 instances tagged `Name: <component>-dev` and `Component: <component>`
- Ansible with `amazon.aws`, `community.general` collections
