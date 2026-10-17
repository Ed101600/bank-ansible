# Bank Ansible

**Configuration management** for hosts and network gear that Terraform does not fully own: bastion hardening, tooling, and optional monitoring collectors.

Terraform in `bank-infra` **creates** AWS resources (including a bastion EC2 if you add one). Ansible **configures** the OS and agents on those hosts. Ansible does **not** replace Terraform for VPC/EKS/RDS, and does **not** replace Argo CD for Kubernetes apps.

## Related repositories

| Repository | Purpose |
| ---------- | ------- |
| [bank-app](https://github.com/Ed101600/bank-app) | Application source and image CI |
| `bank-infra` | Creates VPC, EKS, RDS, bastion instance, IAM |
| `bank-gitops` | Helm + Argo CD + Istio desired state |
| **bank-ansible** (this) | Playbooks/roles for host & agent config |

## When to use Ansible here

| Use Ansible | Do not use Ansible |
| ----------- | ------------------ |
| Bastion packages, users, SSH/SSM hardening | Creating VPCs, EKS, RDS, ECR |
| Installing kubectl/helm on bastion | `kubectl apply` of bank Deployments |
| LogicMonitor (or other) collector on a host | Long-term source of truth for cluster apps |
| Optional Cisco / on-prem device playbooks (later) | Building container images |

Pure EKS with almost no VMs → this repo stays **small** (bastion only). Hybrid estates → grow roles here.

## Layout (to be filled in)

```text
bank-ansible/
├── README.md
├── ansible.cfg
├── inventory/
│   ├── dev.yml              # or dynamic AWS inventory
│   └── prod.yml
├── playbooks/
│   ├── bastion.yml
│   └── collectors.yml       # optional LogicMonitor, etc.
├── roles/
│   ├── common/
│   ├── bastion/
│   └── logicmonitor/        # optional
└── group_vars/
    └── bastion.yml          # non-secret defaults only
```

Secrets (API keys, become passwords) → Ansible Vault, AWS SSM, or CI OIDC — **never** plain Git.

## Prerequisites

- Ansible 2.14+ (or ansible-core pin documented here)  
- Network path to targets (SSM Session Manager preferred over open SSH)  
- Inventory that points at instances Terraform created (tags like `Role=bastion`)  
- IAM/SSH credentials with least privilege  

## Example usage (once playbooks exist)

```sh
# dry run
ansible-playbook -i inventory/dev.yml playbooks/bastion.yml --check

# apply
ansible-playbook -i inventory/dev.yml playbooks/bastion.yml
```

With AWS dynamic inventory (optional later):

```sh
ansible-inventory -i inventory/aws_ec2.yml --graph
```

## Hand-off from Terraform

Document the contract:

1. `bank-infra` outputs bastion instance id / private IP / security group.  
2. Instance profile allows SSM (recommended).  
3. Tags: `Project=bank`, `Role=bastion`, `Env=dev`.  
4. This repo’s inventory selects those tags.  
5. CI (GitHub Actions *or* Jenkins) may run `ansible-playbook` on infra changes — same CI product as `bank-app`, separate workflow.

## LogicMonitor (optional)

If you add LogicMonitor later:

1. Terraform creates (or you designate) a collector host.  
2. Playbook in `roles/logicmonitor` installs/registers the collector.  
3. Dashboards stay in LM; deep service debugging still uses Prometheus/Grafana/ELK from `bank-gitops`.

## Status

- [ ] `ansible.cfg` + inventory for dev  
- [ ] `bastion` role (updates, users, SSM/SSH hardening, basic CLIs)  
- [ ] Documented Terraform tag/output contract  
- [ ] CI workflow to run playbooks (optional)  
- [ ] Optional LogicMonitor collector role  

## License

Learning / personal project. Add a `LICENSE` if you want explicit terms for these playbooks.
