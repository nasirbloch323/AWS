# Secure & Scalable Flask Deployment on AWS VPC using Auto Scaling Group

Complete step-by-step guide with explanations, commands, and troubleshooting — built and tested hands-on.

---

## 📋 Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Step 1: Create VPC](#step-1-create-vpc)
4. [Step 2: Create Subnets](#step-2-create-subnets)
5. [Step 3: Internet Gateway](#step-3-internet-gateway)
6. [Step 4: NAT Gateway](#step-4-nat-gateway)
7. [Step 5: Route Tables](#step-5-route-tables)
8. [Step 6: Launch EC2 Instances (Bastion + Flask)](#step-6-launch-ec2-instances)
9. [Step 7: Install & Run Flask App](#step-7-install--run-flask-app)
10. [Step 8: Target Group](#step-8-target-group)
11. [Step 9: Application Load Balancer](#step-9-application-load-balancer)
12. [Step 10: Create AMI](#step-10-create-ami)
13. [Step 11: Launch Template](#step-11-launch-template)
14. [Step 12: Auto Scaling Group](#step-12-auto-scaling-group)
15. [Step 13: Testing & Validation](#step-13-testing--validation)
16. [Troubleshooting Log (Real Issues Faced)](#troubleshooting-log)
17. [Viva / Interview Q&A Cheat Sheet](#viva-qa-cheat-sheet)

---

## Project Overview

**Industry Problem:** Deploying applications directly in public subnets exposes them to unauthorized access and security risks.

**Solution:** Deploy the backend (Flask app) in **private subnets**, expose only the Load Balancer publicly, and use **NAT Gateway**, **Bastion Host**, and **Auto Scaling** for security, connectivity, and high availability.

**Objective:**
- Isolate application backend in private subnets
- Allow secure SSH access via a Bastion Host
- Enable outbound internet access via NAT Gateway
- Deploy Flask app with Auto Scaling + Application Load Balancer (ALB) for high availability

---

## Architecture Diagram

```
                         Internet
                            │
                    Internet Gateway (IGW)
                            │
        ┌───────────────────┴───────────────────┐
        │                  VPC (10.0.0.0/16)      │
        │                                          │
   ┌────▼────────────┐               ┌────────────▼────┐
   │  Public Subnet 1 │               │ Public Subnet 2  │
   │  10.0.1.0/24      │               │ 10.0.2.0/24       │
   │  - Bastion Host    │               │                    │
   │  - NAT Gateway      │              │  ALB (2nd AZ leg)   │
   │  - ALB (1st AZ leg)  │             │                    │
   └────────┬────────────┘             └─────────┬──────────┘
            │                                     │
     (Application Load Balancer spans both public subnets)
            │
   ┌────────▼────────────┐             ┌─────────▼──────────┐
   │ Private Subnet 1     │             │ Private Subnet 2    │
   │ 10.0.11.0/24          │            │ 10.0.12.0/24          │
   │ - Flask EC2 (ASG)      │           │ - Flask EC2 (ASG)       │
   └────────────────────────┘           └────────────────────────┘

   Private subnets route outbound traffic → NAT Gateway → IGW → Internet
   Auto Scaling Group: Min 1 / Desired 2 / Max 3
```

---

## Step 1: Create VPC

**Console path:** VPC → Your VPCs → Create VPC

| Setting | Value |
|---|---|
| Name tag | `MyFlask-VPC` |
| IPv4 CIDR block | `10.0.0.0/16` |
| DNS Hostnames | Enabled ✅ |
| DNS Resolution | Enabled ✅ |

### Why?
- VPC = your own **isolated private network** inside AWS, like renting a building and deciding what happens on each floor.
- `/16` CIDR gives **65,536 IPs** (10.0.0.0 – 10.0.255.255) — enough room to create multiple subnets later.
- **DNS Hostnames**: lets instances get a readable DNS name, not just an IP.
- **DNS Resolution**: lets the VPC use Amazon's internal DNS to resolve domain names (e.g. accessing `s3.amazonaws.com`).

**Viva Q:** *Why /16 and not /24 for the VPC?*
**A:** The VPC is the big container that will be divided into multiple smaller subnets, so it needs a large address range. Subnets (which are smaller pieces) use smaller ranges like /24.

---

## Step 2: Create Subnets

**Console path:** VPC → Subnets → Create subnet

| # | Name | AZ | CIDR | Auto-assign Public IP |
|---|---|---|---|---|
| 1 | Public-Subnet-1 | AZ-a | `10.0.1.0/24` | ✅ Enabled |
| 2 | Public-Subnet-2 | AZ-b | `10.0.2.0/24` | ✅ Enabled |
| 3 | Private-Subnet-1 | AZ-a | `10.0.11.0/24` | ❌ Disabled |
| 4 | Private-Subnet-2 | AZ-b | `10.0.12.0/24` | ❌ Disabled |

### Why two AZs?
For **High Availability** — if one Availability Zone (like a separate mini data center) goes down, resources in the other AZ keep the app running.

### Why the numbering gap (`.1, .2` then `.11, .12`)?
Not a technical requirement — it's a **design/organization convention**:
- `10.0.1.x`–`10.0.9.x` reserved for Public subnets
- `10.0.11.x`–`10.0.19.x` reserved for Private subnets

This leaves room for future subnets without renumbering everything, and lets anyone reading the CIDR instantly know if a subnet is public or private.

### Public subnet extra step:
Subnet → Actions → **Edit subnet settings** → Enable **"Auto-assign public IPv4 address"**
(Do this ONLY for public subnets — never for private ones.)

---

## Step 3: Internet Gateway

**Console path:** VPC → Internet Gateways → Create internet gateway

1. Name: `MyFlask-IGW`
2. Create → then **Actions → Attach to VPC** → select `MyFlask-VPC`

### Why?
IGW is the **"main door"** connecting your VPC to the internet. Without it, nothing inside the VPC — public or private — can reach or be reached by the internet, no matter what else is configured.

⚠️ One VPC can only have **one** IGW attached.

⚠️ Attaching the IGW is not enough by itself — the **Route Table** must also be told to send traffic to it (Step 5).

---

## Step 4: NAT Gateway

**Console path:** VPC → Elastic IPs → Allocate, then VPC → NAT Gateways → Create

1. Allocate an **Elastic IP** first
2. Create NAT Gateway:
   - Name: `MyFlask-NATGW`
   - Subnet: **Public-Subnet-1** (NAT Gateway always goes in a PUBLIC subnet)
   - Connectivity: Public
   - Elastic IP: the one just allocated

### Why does NAT Gateway sit in a Public Subnet, not Private?
NAT Gateway itself needs internet access (via IGW) to forward traffic on behalf of private instances. IGW only connects to public subnets, so NAT Gateway must live there too.

### What NAT Gateway does:
Gives **private subnet instances outbound-only internet access** (e.g., for `pip install`, OS updates) — but nothing from the internet can initiate a connection INTO the private instance. Traffic direction is one-way (outbound).

**Analogy:** Private subnet server = an employee inside a secure office. To go "shopping" (internet) they go through the receptionist (NAT Gateway) and come back the same way — but no stranger can walk directly to the employee's desk.

### ⚠️ Design trade-off in this project:
Only **1 NAT Gateway** was created (in Public-Subnet-1) to save cost, even though there are 2 AZs. Both private subnets route through this single NAT Gateway.

**Downside:** This creates a **single point of failure** — if AZ-a goes down, Private-Subnet-2 (in AZ-b) also loses internet access, since its NAT Gateway lives in AZ-a.

**Production best practice:** One NAT Gateway per AZ (2 total here) for full redundancy — at extra cost.

---

## Step 5: Route Tables

### Public Route Table
**Console path:** VPC → Route Tables → Create route table

1. Name: `Public-RT`, VPC: `MyFlask-VPC`
2. Add route: `0.0.0.0/0` → Target: **Internet Gateway** (`MyFlask-IGW`)
3. Subnet associations: **Public-Subnet-1** + **Public-Subnet-2**

### Private Route Table
1. Name: `Private-RT`, VPC: `MyFlask-VPC`
2. Add route: `0.0.0.0/0` → Target: **NAT Gateway** (`MyFlask-NATGW`)
3. Subnet associations: **Private-Subnet-1** + **Private-Subnet-2**

### Why?
A Route Table is the **traffic rulebook** — it tells each subnet where to send outbound traffic based on destination.

`0.0.0.0/0` = "any destination on the internet" (catch-all).

- Public-RT → IGW (direct internet access for ALB/Bastion)
- Private-RT → NAT Gateway (outbound-only internet access for Flask servers)

A default **local route** (`10.0.0.0/16 → local`) exists automatically in every route table, allowing all subnets within the VPC to talk to each other.

**Viva Q:** *How do you tell if a subnet is Public or Private just by looking at AWS console?*
**A:** Check its Route Table — if it has a route to an Internet Gateway (`0.0.0.0/0 → igw-xxx`), it's Public. Having "Auto-assign public IP" enabled alone does NOT make a subnet public — the route table is what matters.

---

## Step 6: Launch EC2 Instances

### A) Bastion Host (Public Subnet)

**Security Group `Bastion-SG`:**
| Type | Port | Source |
|---|---|---|
| SSH | 22 | My IP only (`YOUR_IP/32`) |

**Launch instance:**
| Setting | Value |
|---|---|
| Name | `Bastion-Host` |
| AMI | Amazon Linux 2 |
| Instance type | t2.micro |
| Key pair | `bastion-key.pem` (new) |
| Subnet | Public-Subnet-1 |
| Auto-assign Public IP | Enabled |
| Security Group | Bastion-SG |

### B) Flask App Server (Private Subnet)

**Security Group `Flask-SG`:**
| Type | Port | Source |
|---|---|---|
| SSH | 22 | Bastion-SG |
| Custom TCP | 5000 | Bastion-SG (later also ALB-SG) |

**Launch instance:**
| Setting | Value |
|---|---|
| Name | `Flask-App-Server` |
| AMI | Amazon Linux 2023 |
| Instance type | t2.micro |
| Key pair | same `bastion-key.pem` |
| Subnet | Private-Subnet-1 |
| Auto-assign Public IP | **Disabled** |
| Security Group | Flask-SG |

### Why a Bastion Host?
Private subnet instances have no public IP, so you can't SSH into them directly. The Bastion is a **"jump server"** — public, reachable, and used as a stepping stone:

```
Local Computer → SSH → Bastion Host (Public) → SSH → Flask Server (Private)
```

### Why source = Security Group name, not IP, in Flask-SG?
Referencing another Security Group as the source (instead of a raw IP) means "allow traffic from any instance using that SG" — more secure and flexible than hardcoding IPs that can change.

---

## Step 7: Install & Run Flask App

Copy the key to Bastion (from local machine):
```bash
scp -i bastion-key.pem bastion-key.pem ec2-user@<Bastion_Public_IP>:/home/ec2-user/
```

SSH into Bastion:
```bash
ssh -i bastion-key.pem ec2-user@<Bastion_Public_IP>
chmod 400 bastion-key.pem
```

SSH from Bastion into the Private Flask server:
```bash
ssh -i bastion-key.pem ec2-user@<Private_Instance_IP>
```

On the Private server:
```bash
sudo dnf update -y
sudo dnf install -y python3
pip3 --version
# if pip3 missing:
python3 -m ensurepip --upgrade

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
```

**Run in foreground (for first test):**
```bash
python3 app.py
```

**Run properly in background (so it survives SSH disconnects):**
```bash
nohup python3 app.py > flask.log 2>&1 &
```

### Why `host='0.0.0.0'` and not `127.0.0.1`?
`0.0.0.0` makes the app listen on **all network interfaces**, so it's reachable from outside the machine (e.g. from the Load Balancer). `127.0.0.1` (localhost) only accepts connections from within that exact machine.

### Why `nohup ... &`?
- `nohup` = "no hang up" — keeps the process alive even after the SSH session closes.
- `&` = runs the command in the background so the terminal is free.
- `> flask.log 2>&1` = saves output/errors to a log file instead of the terminal.

**Test it (from Bastion, in a separate session):**
```bash
curl http://<Private_Instance_IP>:5000
# Expected: Hello from Private Subnet!
```

---

## Step 8: Target Group

**Console path:** EC2 → Target Groups → Create target group

| Setting | Value |
|---|---|
| Target type | Instances |
| Name | `Flask-TG` |
| Protocol | HTTP |
| Port | 5000 |
| VPC | MyFlask-VPC |
| Health check path | `/` |

Register the `Flask-App-Server` instance on port `5000`.

### Why?
A Target Group is a **list of servers to send traffic to**. The ALB doesn't know individual servers directly — it always routes through a Target Group.

### Health Checks
The ALB regularly pings the health check path (`/`). A `200 OK` response = **Healthy**; no response = **Unhealthy** (traffic stops going there).

---

## Step 9: Application Load Balancer

**Security Group `ALB-SG`:**
| Type | Port | Source |
|---|---|---|
| HTTP | 80 | `0.0.0.0/0` (public-facing, open to everyone) |

**Create ALB:**
| Setting | Value |
|---|---|
| Name | `Flask-ALB` |
| Scheme | Internet-facing |
| IP type | IPv4 |
| VPC | MyFlask-VPC |
| Subnets | Public-Subnet-1 + Public-Subnet-2 (both AZs) |
| Security Group | ALB-SG |
| Listener | HTTP:80 → forward to `Flask-TG` |

### ⚠️ Critical extra step — update Flask-SG:
Add a new inbound rule so the ALB can actually reach the Flask app:
| Type | Port | Source |
|---|---|---|
| Custom TCP | 5000 | **ALB-SG** |

Without this, the Target Group will show **Unhealthy** even if the app is running fine — this was the single most common issue encountered during testing.

### Concepts, simplified:

**Target Group** = a list/register of which servers ("waiters") can serve traffic.
**ALB** = the manager/receptionist deciding which waiter serves which customer.

Flow:
```
Internet User → Port 80 → ALB (checks ALB-SG: 0.0.0.0/0 allowed)
   → Listener forwards to Flask-TG
   → Target Group checks health
   → Flask Server on port 5000 (Flask-SG allows ALB-SG only)
   → Response back to User
```

---

## Step 10: Create AMI

Once the manual Flask instance is tested and working:

**Console path:** EC2 → Instances → select `Flask-App-Server` → Actions → Image and templates → **Create image**

| Setting | Value |
|---|---|
| Image name | `Flask-App-AMI` |
| No reboot | ✅ (optional, keeps instance running while imaging) |

Wait for status: Pending → **Available**.

### Why?
An AMI is a **complete snapshot** of the server — OS, installed packages (Python, Flask), and files (`app.py`). Auto Scaling Group needs this as a blueprint to create identical new servers automatically, without manually reinstalling anything.

---

## Step 11: Launch Template

**Console path:** EC2 → Launch Templates → Create launch template

| Setting | Value |
|---|---|
| Name | `Flask-Launch-Template` |
| AMI | My AMIs → `Flask-App-AMI` |
| Instance type | t2.micro |
| Key pair | `bastion-key.pem` |
| Security Group | Flask-SG |
| Subnet | *not selected here* — ASG decides this later |

**Advanced details → User data:**
```bash
#!/bin/bash
cd /home/ec2-user
nohup python3 app.py > flask.log 2>&1 &
```

### Why User Data?
This script runs **automatically the first time a new instance boots**. It starts the Flask app without any manual SSH/setup — essential for Auto Scaling to work unattended.

---

## Step 12: Auto Scaling Group

**Console path:** EC2 → Auto Scaling Groups → Create Auto Scaling group

| Setting | Value |
|---|---|
| Name | `Flask-ASG` |
| Launch template | Flask-Launch-Template |
| VPC | MyFlask-VPC |
| Subnets | Private-Subnet-1 + Private-Subnet-2 |
| Load balancing | Attach to existing target group → `Flask-TG` |
| Health checks | ELB health checks: **enabled** |
| Desired capacity | 2 |
| Minimum capacity | 1 |
| Maximum capacity | 3 |

### Why Min 1 / Desired 2 / Max 3?
- **Minimum (1):** At least 1 server always running, no matter how low traffic is.
- **Desired (2):** Normal target the ASG tries to maintain.
- **Maximum (3):** Ceiling — won't scale beyond this even under heavy load.

### Why enable ELB health checks (not just EC2 health checks)?
EC2-level checks only know if the *instance* is running. ELB health checks verify the *application* is actually responding — catching app-level crashes that EC2 checks would miss, and triggering automatic replacement.

**Analogy:** ASG is a restaurant manager who calls in more waiters when it's busy (up to Max), and instantly replaces a waiter who falls sick (crash) — without the customer noticing.

---

## Step 13: Testing & Validation

1. **Confirm new instances launched:** EC2 → Instances → should see 2 new instances in Private subnets, state = Running.
2. **Target Group health:** Target Groups → Flask-TG → Targets tab → should show Healthy within a couple minutes.
3. **Auto-healing test (best viva demo):**
   - Manually terminate one ASG instance (Actions → Instance state → Terminate)
   - Wait 1–2 minutes
   - Check Auto Scaling Groups → Flask-ASG → Activity tab — a replacement instance should launch automatically to restore Desired Capacity.
4. **End-to-end test:** Open the ALB DNS name in a browser repeatedly — should always return `Hello from Private Subnet!` regardless of which backend instance served it.

---

## Troubleshooting Log (Real Issues Faced)

| Issue | Root Cause | Fix |
|---|---|---|
| SSH from Bastion to Private instance hangs/times out | Flask-SG didn't allow SSH (port 22) from Bastion-SG | Add inbound rule: Port 22, Source = Bastion-SG |
| Website not reachable via private IP/localhost in browser | Private IPs and `127.0.0.1` are not internet-routable / not "this machine" from browser's perspective | Only test private IP via `curl` from inside the VPC (Bastion); public access only works through ALB |
| Target Group shows Unhealthy | 1) Flask app not running (SSH session closed, killed foreground process) 2) Flask-SG missing rule allowing ALB-SG on port 5000 | Restart app with `nohup ... &`; add ALB-SG inbound rule on port 5000 |
| Flask app stops randomly | App was run in the foreground; closing/losing the SSH session killed it | Always run with `nohup python3 app.py > flask.log 2>&1 &` |
| Browser shows "This site can't be reached" even though `curl` from CMD works fine | Browser (Chrome) auto-upgrading `http://` to `https://`, but ALB only has an HTTP:80 listener, no HTTPS:443 listener | Clear HSTS cache via `chrome://net-internals/#hsts`, or test via `curl` in terminal to confirm infra is fine, or use another browser |
| All targets (including previously-healthy manual instance) suddenly Unhealthy | App process had stopped again after SSH session closed | Re-verify `ps aux | grep app.py` on each instance; restart with `nohup` |

**Key lesson:** When a Target Group is Unhealthy, always check in this order:
1. Is the app actually running? (`ps aux`)
2. Does `curl localhost:5000` work on the instance itself?
3. Is the Security Group allowing the ALB-SG on the app's port?
4. Is the Target Group's registered port correct?

---

## Viva / Interview Q&A Cheat Sheet

**Q: Why deploy the backend in a private subnet instead of public?**
A: Public subnets are internet-reachable, exposing the backend to unauthorized access. Keeping it private means only the ALB (via Security Group rules) can reach it — this is the core security best practice this whole project demonstrates.

**Q: What's the difference between an Internet Gateway and a NAT Gateway?**
A: IGW allows two-way traffic (in and out) and is used by public subnet resources. NAT Gateway allows outbound-only traffic for private subnet resources — nothing from the internet can initiate a connection to a private instance through it.

**Q: Why does a Bastion Host exist?**
A: Since private instances have no public IP, the Bastion (in a public subnet) acts as a secure "jump server" for SSH access into private resources.

**Q: What's the difference between a Target Group and a Load Balancer?**
A: The ALB receives traffic and decides where to route it. The Target Group is just the list of servers that traffic can be routed to. ALB = manager, Target Group = employee register.

**Q: Why is a health check important?**
A: So the Load Balancer never sends traffic to a crashed/unresponsive server — only "Healthy" targets receive traffic.

**Q: Explain Min/Desired/Max in Auto Scaling.**
A: Min = floor (never go below), Desired = normal target size, Max = ceiling (never exceed, even under heavy load).

**Q: What is an AMI and why is it needed before Auto Scaling?**
A: An AMI is a full snapshot of a working, tested server (OS + installed software + app files). The Auto Scaling Group uses it as a blueprint via the Launch Template to create identical new servers automatically.

**Q: What does User Data do in a Launch Template?**
A: It's a script that runs automatically the first time a new instance boots — used here to auto-start the Flask app so no manual setup is needed for new Auto Scaling instances.

**Q: What happens if an instance crashes?**
A: The Auto Scaling Group detects it (via ELB health checks) and automatically launches a replacement instance using the Launch Template, maintaining the Desired Capacity — this is called self-healing.

**Q: What happens if an entire Availability Zone goes down?**
A: Since resources are spread across 2 AZs, the other AZ's instances continue serving traffic. The ALB automatically routes only to Healthy targets, regardless of AZ.

**Q: Why did you use only 1 NAT Gateway instead of 2 (one per AZ)?**
A: To reduce cost during learning/testing — NAT Gateway has hourly + data transfer charges. In production, best practice is one NAT Gateway per AZ to avoid a single point of failure.

**Q: Why `host='0.0.0.0'` in the Flask app instead of `127.0.0.1`?**
A: `0.0.0.0` makes the app listen on all network interfaces so it can be reached from outside the machine (e.g. by the ALB's health checks). `127.0.0.1` only accepts local traffic.

---

## Final Architecture Summary

```
Internet User
    │
    ▼
ALB (Public Subnets, Internet-facing, port 80)
    │
    ▼
Target Group (health checks on path "/")
    │
    ▼
Auto Scaling Group (Min:1 / Desired:2 / Max:3)
    │
    ▼
Flask EC2 Servers (Private Subnets, port 5000, no public IP)
    │
    ▼ (outbound internet only)
NAT Gateway (Public Subnet) → Internet Gateway → Internet
```

**Security layers:**
- Bastion Host — SSH admin access only, restricted to a specific IP
- Flask Servers — no public IP, only accept traffic from ALB-SG and Bastion-SG
- ALB — the only public-facing entry point (HTTP:80)
