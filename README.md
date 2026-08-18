# AWS VPC Setup with Load Balancer — Secure Flask App Deployment

A hands-on DevOps project demonstrating a **production-style, secure AWS network architecture** with public/private subnet isolation, a Bastion Host for admin access, and an Application Load Balancer (ALB) exposing a private Flask application to the internet.

---

## 📌 Project Overview

This project deploys a Python Flask API inside a **private subnet** (no direct internet exposure) and makes it accessible to end users **only** through an Application Load Balancer. Administrative access to the private instance is possible **only** through a Bastion Host, following the principle of least privilege.

**Key idea:** the backend is never directly reachable from the internet — every path in or out goes through a controlled gateway.

---

## 🏗️ Architecture

```
                            Internet
                               │
                        ┌──────┴──────┐
                        │   Internet   │
                        │   Gateway    │
                        └──────┬──────┘
                               │
                    ┌──────────┴──────────┐
                    │     Public Subnet    │  10.0.1.0/24
                    │  ┌────────┐ ┌──────┐ │
                    │  │Bastion │ │ ALB  │ │
                    │  │ Host   │ │      │ │
                    │  └───┬────┘ └───┬──┘ │
                    │      │  NAT GW  │    │
                    └──────┼─────┬────┼────┘
                           │     │    │
                    ┌──────┼─────┼────┼────┐
                    │    Private Subnet     │  10.0.2.0/24
                    │      │              │ │
                    │  ┌───▼──────────────▼┐│
                    │  │  Flask App Server  ││
                    │  │   (port 5000)      ││
                    │  └────────────────────┘│
                    └────────────────────────┘

VPC CIDR: 10.0.0.0/16
```

**Traffic flow:**
- **Users →** ALB (public) → Target Group → Flask App (private, port 5000)
- **Admin →** SSH → Bastion Host (public) → SSH → Flask Server (private)
- **Flask Server →** NAT Gateway (public subnet) → Internet (outbound only, for package updates)

---

## 🧰 Components & Why Each One Is Used

| Component | Purpose |
|---|---|
| **VPC** (`10.0.0.0/16`) | Isolated private network for all project resources |
| **Public Subnet** (`10.0.1.0/24`) | Hosts internet-facing resources: Bastion Host, NAT Gateway, ALB |
| **Private Subnet** (`10.0.2.0/24`) | Hosts the Flask app server; no public IP, no direct inbound internet access |
| **Internet Gateway (IGW)** | Two-way connection between the VPC and the internet; used by the public subnet |
| **NAT Gateway** | Lives in the public subnet; gives the private subnet **outbound-only** internet access (e.g., `dnf update`) without exposing it to inbound traffic |
| **Route Tables** | Public RT → routes `0.0.0.0/0` to IGW. Private RT → routes `0.0.0.0/0` to NAT Gateway |
| **Bastion Host** | Public EC2 instance used purely as a secure SSH jump box into the private subnet |
| **Flask App Server** | Private EC2 instance running the actual application on port 5000 |
| **Security Groups** | Per-instance firewalls enforcing least-privilege access (see below) |
| **Application Load Balancer (ALB)** | Public entry point that forwards user traffic to the Flask server via a Target Group |
| **Target Group** | Tells the ALB which instance(s) and port to forward traffic to |

---

## 🔐 Security Group Rules (Corrected)

> ⚠️ **Fix note:** The original design document had inconsistent ports (mixing 5000/80) and an overly-open ALB rule. Corrected rules below.

**Bastion Host SG**
| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | SSH | 22 | Your IP `/32` |
| Outbound | All traffic | All | 0.0.0.0/0 |

**Flask App Server SG**
| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | SSH | 22 | Bastion Host SG |
| Inbound | HTTP | 5000 | ALB SG |
| Outbound | All traffic | All | 0.0.0.0/0 |

**ALB SG**
| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | HTTP | 80 | 0.0.0.0/0 |
| Outbound | HTTP | 5000 | Flask App Server SG |

*(Original doc had the Flask server allowing inbound directly from the Bastion on port 5000 and the ALB SG open on port 5000 from the internet — both were mistakes. SSH between Bastion↔Flask stays on 22; app traffic between ALB↔Flask stays on 5000; users only ever hit the ALB on 80.)*

---

## 🚀 Step-by-Step Deployment

### 1. Create VPC
- CIDR: `10.0.0.0/16`
- Enable DNS Hostnames + DNS Resolution

### 2. Create Subnets
- `Public-Subnet` — `10.0.1.0/24` — auto-assign public IP: **enabled**
- `Private-Subnet` — `10.0.2.0/24` — auto-assign public IP: **disabled**

### 3. Internet Gateway
- Create IGW → attach to VPC

### 4. NAT Gateway
- Allocate an Elastic IP
- Create NAT Gateway **in the Public Subnet**, attach the Elastic IP

### 5. Route Tables
- `Public-RT`: `0.0.0.0/0 → IGW`, associate with Public Subnet
- `Private-RT`: `0.0.0.0/0 → NAT Gateway`, associate with Private Subnet

### 6. Launch EC2 Instances
**Bastion Host** (Public Subnet)
- AMI: Amazon Linux 2
- Type: t2.micro
- Key pair: `nasir.pem`
- SG: SSH (22) from your IP only

**Flask App Server** (Private Subnet)
- AMI: Amazon Linux 2 *(kept consistent — see Fixes section)*
- Type: t2.micro
- Key pair: `nasir.pem` (same key, copied over via `scp`)
- SG: SSH (22) from Bastion SG, HTTP (5000) from ALB SG

### 7. Connect to the Private Instance
```bash
# Copy key from local machine to Bastion
scp -i "nasir.pem" nasir.pem ec2-user@<BASTION_PUBLIC_DNS>:/home/ec2-user/

# SSH into Bastion
ssh -i "nasir.pem" ec2-user@<BASTION_PUBLIC_DNS>

# From inside Bastion, set permissions and hop to the private instance
chmod 400 nasir.pem
ssh -i "nasir.pem" ec2-user@10.0.2.120
```

### 8. Deploy Flask App (on the private instance)
```bash
sudo dnf update -y
sudo dnf install -y python3
pip3 install flask

cat <<EOF > app.py
from flask import Flask
app = Flask(__name__)

@app.route('/')
def home():
    return "Hello from Private Subnet!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

python3 app.py
```

### 9. Configure the Application Load Balancer
- Type: Application Load Balancer, Internet-facing
- Listener: HTTP on port **80**
- Subnets: attach the Public Subnet(s)
- Target Group: HTTP, target type = Instance, port **5000**, register the Flask instance
- Health check path: `/`

### 10. Validate
- Open the ALB's DNS name in a browser
- You should see: `Hello from Private Subnet!`

---

## 🩹 Fixes Applied vs. Original Design Doc

1. **Key pair naming** — original doc referenced three different keys (`bastion-key.pem`, `ansible-master.pem`, `my-key.pem`). Standardized to a single key (`nasir.pem`) for both Bastion and Flask instances.
2. **AMI inconsistency** — original mixed Amazon Linux 2, Amazon Linux 2023, and Ubuntu across steps. Standardized to **Amazon Linux 2** for both instances.
3. **ALB Security Group** — original allowed inbound on port 5000 from `0.0.0.0/0`, which exposes the app port directly and bypasses the whole point of the ALB. Fixed: ALB listens on port **80** from the internet, and forwards to the Flask target on port 5000 internally.
4. **Flask server inbound rule** — original had "Allow HTTP (Port 5000) only from Bastion Host," which doesn't make sense (Bastion only needs SSH, not app traffic). Fixed: Flask server accepts SSH (22) from Bastion SG and HTTP (5000) from ALB SG only.
5. **Private instance connectivity** — clarified that EC2 Instance Connect (browser SSH) does **not** work on private-subnet instances by design; access must go through the Bastion Host or an EC2 Instance Connect Endpoint.

---

## 🔒 Security Summary
- Flask App Server has **no public IP** and is never directly internet-reachable.
- Admin access is possible **only** via the Bastion Host.
- End users reach the app **only** via the ALB.
- NAT Gateway allows the private instance to reach the internet **outbound only** (e.g., for OS/package updates).

---

## 🛣️ Possible Extensions
- Auto Scaling Group for the Flask server across multiple AZs
- RDS database in a dedicated private (data) subnet
- HTTPS listener on the ALB with an ACM certificate
- CloudWatch monitoring + alarms
- Infrastructure as Code (Terraform/CloudFormation) instead of manual console steps

---

## 🧑‍💻 Author
Nasir — DevOps Project (Day 91)
