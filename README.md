# AWS Custom VPC — Public & Private Subnet Architecture

A hands-on implementation of a production-style AWS VPC, built from scratch to understand core cloud networking concepts used in real-world DevOps environments.

## 📌 Overview

This project sets up an isolated Virtual Private Cloud (VPC) with proper network segmentation — separating internet-facing resources from internal/private resources, following AWS best practices.

## 🏗️ Architecture

```
                         Internet
                            │
                    ┌───────▼────────┐
                    │ Internet Gateway│
                    └───────┬────────┘
                            │
        ┌───────────────────────────────────────┐
        │                  VPC                   │
        │            172.16.0.0/16                │
        │                                          │
        │  ┌─────────────────┐  ┌───────────────┐ │
        │  │  Public Subnet   │  │ Private Subnet│ │
        │  │  172.16.0.0/24   │  │ 172.16.1.0/24 │ │
        │  │                  │  │               │ │
        │  │   EC2 (Web)      │  │  EC2 (DB)     │ │
        │  │   Public IP      │  │  Private IP   │ │
        │  └────────┬─────────┘  └───────┬───────┘ │
        │           │                    │         │
        │           │            ┌───────▼──────┐  │
        │           │            │ NAT Gateway  │  │
        │           │            └───────┬──────┘  │
        │           └────────────────────┘         │
        └───────────────────────────────────────────┘
```

## ⚙️ Components

| Component | Purpose |
|---|---|
| **VPC** | Isolated private network (`172.16.0.0/16`) |
| **Public Subnet** | Hosts internet-facing resources (web servers, bastion host) |
| **Private Subnet** | Hosts internal resources (databases, backend services) |
| **Internet Gateway (IGW)** | Provides two-way internet access to the public subnet |
| **NAT Gateway** | Provides one-way (outbound-only) internet access to the private subnet |
| **Route Tables** | Controls traffic flow between subnets, IGW, and NAT Gateway |
| **Bastion Host** | Public EC2 used as a secure jump server to access private EC2 instances |

## 🚀 Setup Steps

1. **Create the VPC**
   - CIDR block: `172.16.0.0/16`

2. **Create Subnets**
   - Public subnet: `172.16.0.0/24`
   - Private subnet: `172.16.1.0/24`

3. **Create & Attach Internet Gateway**
   - Attach the IGW to the VPC

4. **Create NAT Gateway**
   - Deploy inside the **public subnet**
   - Allocate an Elastic IP

5. **Configure Route Tables**
   - Public route table → `0.0.0.0/0` → Internet Gateway
   - Private route table → `0.0.0.0/0` → NAT Gateway
   - Associate each route table with its respective subnet

6. **Launch EC2 Instances**
   - Public EC2: auto-assign public IP enabled
   - Private EC2: no public IP

## ✅ Verification

**Public EC2 → Internet (via IGW)**
```bash
ssh -i "my-key.pem" ec2-user@<PUBLIC_EC2_PUBLIC_IP>
ping google.com
```

**Private EC2 → Internet (via NAT Gateway, using public EC2 as bastion)**
```bash
ssh -A -i "my-key.pem" ec2-user@<PUBLIC_EC2_PUBLIC_IP>
ssh ec2-user@<PRIVATE_EC2_PRIVATE_IP>
ping google.com
```

Both connectivity tests passed, confirming the Internet Gateway and NAT Gateway are routing traffic correctly.

## 🔐 Security Notes

- Private EC2's Security Group only allows SSH (port 22) from the public EC2's Security Group — not from the open internet.
- Private subnet has no direct inbound path from the internet; all outbound traffic is routed through the NAT Gateway.

## 🛠️ Tech Stack

- **AWS VPC**
- **AWS EC2**
- **AWS Internet Gateway / NAT Gateway**
- **AWS Route Tables & Security Groups**

## 📈 Next Steps

- [ ] Automate this setup using **Terraform**
- [ ] Add **Multi-AZ** deployment for high availability
- [ ] Implement stricter **NACLs**
- [ ] Set up a proper **Bastion Host** with restricted access rules

## 👤 Author

**Nasir Mehmood**
DevOps enthusiast, learning cloud infrastructure hands-on.
