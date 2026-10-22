# AWS VPC Peering — Private Multi-VPC Connectivity

**EWB Cloud Labs | AWS Architect Associate | Oct 2026**  
**Student:** Akash Mondal &nbsp;|&nbsp; **Region:** ap-south-1 (Mumbai)

[![AWS](https://img.shields.io/badge/AWS-VPC%20Peering-FF9900?logo=amazon-aws&logoColor=white)](https://aws.amazon.com/vpc/)
[![IaC](https://img.shields.io/badge/IaC-Terraform%20%26%20CloudFormation-623CE4?logo=terraform&logoColor=white)](https://www.terraform.io/)
[![Status](https://img.shields.io/badge/Status-Completed-brightgreen)](https://github.com/go2akash/EWB-Courses-AWS-VPC-Peering-Student-Project)

---

## What This Project Does

This project connects two isolated AWS VPCs using a **VPC Peering Connection** and proves that private instances in each VPC can communicate with each other using only their private IP addresses — no public internet involved.

The setup uses a **Bastion host** in VPC A as the only entry point, with **SSH ProxyJump** to reach private instances in both VPCs.

---

## Architecture

```
Your Laptop
    │
    │ SSH (Port 22)
    ▼
Internet Gateway (ewb-igw-a)
    │
    ▼
┌─────────────────────────────────────┐
│  VPC A  (10.10.0.0/16)             │
│                                     │
│  Public Subnet (10.10.1.0/24)      │
│  └── ewb-bastion  [10.10.1.10]     │
│                                     │
│  Private Subnet (10.10.2.0/24)     │
│  └── ewb-app-a   [10.10.2.10]     │
└──────────────┬──────────────────────┘
               │ VPC Peering (ewb-a-b-peer)
               │ Private AWS Backbone
┌──────────────┴──────────────────────┐
│  VPC B  (10.20.0.0/16)             │
│                                     │
│  Private Subnet (10.20.1.0/24)     │
│  └── ewb-app-b   [10.20.1.10]     │
│      (No IGW, No Public IP)        │
└─────────────────────────────────────┘
```

---

## Resources Created

| Resource | Name | CIDR / IP |
|---|---|---|
| VPC A | `ewb-vpc-a` | `10.10.0.0/16` |
| Public Subnet | `ewb-a-public` | `10.10.1.0/24` |
| Private Subnet A | `ewb-a-private` | `10.10.2.0/24` |
| VPC B | `ewb-vpc-b` | `10.20.0.0/16` |
| Private Subnet B | `ewb-b-private` | `10.20.1.0/24` |
| Internet Gateway | `ewb-igw-a` | Attached to VPC A |
| Bastion Host | `ewb-bastion` | `10.10.1.10` + Public IP |
| App Server A | `ewb-app-a` | `10.10.2.10` (no public IP) |
| App Server B | `ewb-app-b` | `10.20.1.10` (no public IP) |
| Peering Connection | `ewb-a-b-peer` | VPC A ↔ VPC B |

---

## Route Tables

| Route Table | Subnet | Routes Added |
|---|---|---|
| `ewb-rt-a-public` | ewb-a-public | `0.0.0.0/0 → ewb-igw-a`, `10.20.0.0/16 → pcx-ID` |
| `ewb-rt-a-private` | ewb-a-private | `10.20.0.0/16 → pcx-ID` |
| `ewb-rt-b-private` | ewb-b-private | `10.10.0.0/16 → pcx-ID` |

> The return route in `ewb-rt-b-private` is critical — without it, reply packets from App-B have no path back and the connection fails silently.

---

## Tests & Results

| Test | Command | Result |
|---|---|---|
| Baseline failure (before peering) | `ping -c 4 10.20.1.10` from App-A | `100% packet loss` ✅ |
| Forward path | `ping -c 4 10.20.1.10` from App-A | `0% loss` ✅ |
| Reverse path | `ping -c 4 10.10.2.10` from App-B | `0% loss` ✅ |
| SSH via ProxyJump | `ssh ewb-app-b` from laptop | Shell on App-B ✅ |
| Zero public IP | EC2 console check | No public IP on App-A or App-B ✅ |
| Controlled failure | Delete ICMP rule on App-B SG | `100% packet loss` ✅ |
| Recovery | Re-add ICMP rule | `0% loss` instantly ✅ |
| Cleanup | Terminate EC2s, delete peering | All resources removed ✅ |

---

## Repository Structure

```
├── Akash_Mondal_EWB_VPC_Peering.pdf   ← Submission report PDF
├── README.md
├── MANUAL_CONSOLE_GUIDE.md             ← Step-by-step AWS Console walkthrough
├── ssh_setup_guide.md                  ← SSH ProxyJump setup (Windows/Mac/Linux)
├── cloudformation/
│   └── vpc_peering_lab.yaml            ← 1-click CloudFormation deployment
├── terraform/
│   ├── main.tf                         ← Full infrastructure as code
│   ├── variables.tf
│   └── outputs.tf
└── screenshots/
    ├── checkpoints/                    ← 18 milestone verification screenshots
    └── *.png                           ← Full execution capture (43 images)
```

---

## Deploy It Yourself

### Option A — CloudFormation (fastest, ~3 mins)

1. Go to AWS Console → **CloudFormation → Create Stack → Upload template**
2. Upload `cloudformation/vpc_peering_lab.yaml`
3. Set parameters: Stack name `ewb-vpc-peering-stack`, KeyName `ewb-lab`, your public IP as `/32`
4. Submit — done when status shows `CREATE_COMPLETE`

### Option B — Terraform

```bash
cd terraform
terraform init
terraform plan -var="admin_ip=YOUR.IP.HERE/32"
terraform apply -var="admin_ip=YOUR.IP.HERE/32"
```

### Option C — Manual Console

Follow the step-by-step guide: 👉 [MANUAL_CONSOLE_GUIDE.md](MANUAL_CONSOLE_GUIDE.md)

---

## SSH Access Setup

```bash
# macOS/Linux
chmod 400 ~/.ssh/ewb-lab.pem

# ~/.ssh/config
Host ewb-bastion
    HostName <BASTION_PUBLIC_IP>
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem

Host ewb-app-a
    HostName 10.10.2.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    ProxyJump ewb-bastion

Host ewb-app-b
    HostName 10.20.1.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    ProxyJump ewb-bastion
```

Full guide → [ssh_setup_guide.md](ssh_setup_guide.md)

---

## Key Learnings

- VPC Peering is **non-transitive** — A↔B and B↔C does not give A↔C
- Both sides need route table entries — forward **and** return routes are mandatory
- Security Groups are **stateful** but still need explicit inbound rules per protocol
- SSH ProxyJump means App-B sees the Bastion's private IP (`10.10.1.10`) as the source, not your laptop
- Active peering status alone is not enough — routes + SG rules must also align

---

## Cleanup

Terminate `ewb-bastion`, `ewb-app-a`, `ewb-app-b` → delete peering → delete security groups → delete subnets & route tables → detach & delete `ewb-igw-a` → delete both VPCs.

---

*EWB Cloud Labs — Learn. Elevate. Empower.*
