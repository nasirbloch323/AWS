# 🔐 Secure Multi-VPC Deployment: Flask App + Jenkins with VPC Peering on AWS

A complete, beginner-friendly, real-world DevOps project that shows how to deploy a **Flask application** in a private subnet and a **Jenkins CI/CD server** in a separate VPC, then securely connect both environments using **AWS VPC Peering**.

This README explains everything from scratch — even if you're a complete beginner, you should be able to follow along step by step.

---

## 👤 Author

**Nasir Mehmood** — DevOps Engineer
- 🔗 LinkedIn: [linkedin.com/in/nasirbloch323](https://www.linkedin.com/in/nasirbloch323)
- 🐙 GitHub: [github.com/nasirbloch323](https://github.com/nasirbloch323)

---

## 📖 Project Overview

In real companies, infrastructure is never kept in one single network. It is split into isolated environments for **security**, **scalability**, and **better access control**.

This project demonstrates that exact pattern:

| Component | Purpose | Location |
|---|---|---|
| Flask App | The actual application | Private subnet (Application VPC) |
| Jenkins | CI/CD automation tool | Jenkins VPC (separate network) |
| Bastion Host | Secure gateway to reach private servers | Public subnet (Application VPC) |
| VPC Peering | Private, low-latency link between the two VPCs | Between both VPCs |

---

## 🏗️ Architecture

```
                          AWS REGION
 ┌───────────────────────────┐        ┌───────────────────────────┐
 │      Application VPC       │        │        Jenkins VPC         │
 │      (10.0.0.0/16)          │◄──────►│      (192.168.0.0/16)       │
 │                             │  VPC   │                             │
 │  Public Subnet              │ Peering│  Public Subnet              │
 │   └─ Bastion Host           │        │   └─ Internet Gateway       │
 │                             │        │   └─ Jenkins EC2            │
 │  Private Subnet             │        │                             │
 │   └─ Flask App (port 5000)  │        │                             │
 └───────────────────────────┘        └───────────────────────────┘
```

**Flow:** Jenkins (in its own VPC) reaches the Flask app running in a private subnet of another VPC — securely, without exposing the app to the public internet — using VPC Peering.

---

## ✅ Prerequisites

- An AWS account
- Basic knowledge of SSH
- A `.pem` key pair created in AWS
- AWS Console access (EC2, VPC sections)

---

## 🪜 Step-by-Step Implementation

### Step 1: Create Two VPCs

**1.1 Application VPC**
- CIDR: `10.0.0.0/16`
- Subnets:
  - Public: `10.0.1.0/24`
  - Private: `10.0.2.0/24`

**1.2 Jenkins VPC**
- CIDR: `192.168.0.0/16`
- Subnet:
  - Public: `192.168.1.0/24`

> Go to **AWS Console → VPC → Create VPC**, and repeat for both.

---

### Step 2: Create Internet Gateway (IGW) and NAT Gateway

**2.1 Attach Internet Gateway**
For **both** VPCs:
- Create an Internet Gateway
- Attach it to the VPC (this allows public subnets to reach the internet)

**2.2 NAT Gateway (Application VPC only)**
- Allocate an **Elastic IP**
- Create a **NAT Gateway** inside the public subnet (`10.0.1.0/24`)
- This lets the **private subnet** (Flask server) reach the internet for updates/installs, without being publicly exposed

---

### Step 3: Configure Route Tables

**Application VPC**
| Subnet | Destination | Target |
|---|---|---|
| Public (`10.0.1.0/24`) | `0.0.0.0/0` | Internet Gateway |
| Private (`10.0.2.0/24`) | `0.0.0.0/0` | NAT Gateway |

**Jenkins VPC**
| Subnet | Destination | Target |
|---|---|---|
| Public (`192.168.1.0/24`) | `0.0.0.0/0` | Internet Gateway |

---

### Step 4: Set Up VPC Peering

1. Go to **VPC Dashboard → Peering Connections → Create Peering Connection**
   - VPC A: Application VPC
   - VPC B: Jenkins VPC
2. Accept the peering request (in the same or requester account).
3. Update route tables on **both sides**:
   - Application VPC → add route: `192.168.0.0/16` → Peering Connection
   - Jenkins VPC → add route: `10.0.0.0/16` → Peering Connection
4. Update **Security Groups** on both sides to allow traffic (e.g. TCP port `5000` for Flask).

> Without this step, the two networks cannot "see" each other even though they're peered.

---

### Step 5: Launch EC2 Instances

**a) Bastion Host** (Public Subnet – Application VPC)
- AMI: Amazon Linux 2
- Type: `t2.micro`
- Key pair: `bastion-key.pem`
- Security Group: allow **SSH (22)** only from **your IP**

**b) Flask App Server** (Private Subnet – Application VPC)
- AMI: Amazon Linux 2
- Type: `t2.micro`
- Key pair: same `bastion-key.pem`
- Security Group:
  - Allow **SSH (22)** only from Bastion Host's Security Group
  - Allow **HTTP (5000)** only from Bastion Host / Jenkins SG

**c) Jenkins Server** (Public Subnet – Jenkins VPC)
- AMI: Ubuntu (or Amazon Linux)
- Type: `t2.micro` / `t2.medium` recommended
- Key pair: your Jenkins key
- Security Group: allow **SSH (22)** and **8080** from your IP

**Copy your key to a server (if needed for internal hops):**
```bash
scp -i bastion-key.pem bastion-key.pem ubuntu@<EC2_PUBLIC_IP>:/home/ubuntu/
```

---

### Step 6: Install and Run Flask App (Private Server)

**Access the private server through the Bastion Host:**
```bash
# From your local machine → into Bastion Host
ssh -i bastion-key.pem ec2-user@<Bastion_Public_IP>

# From Bastion Host → into the private Flask server
ssh -i bastion-key.pem ec2-user@<Private_Instance_IP>
```

**Install Python3 and pip3 (Amazon Linux 2023):**
```bash
sudo dnf update -y
sudo dnf install -y python3
pip3 --version
```

If `pip3` is missing:
```bash
python3 -m ensurepip --upgrade
```

**Install Flask:**
```bash
pip3 install flask
```

**Create a simple Flask app:**
```bash
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

**Run it:**
```bash
python3 app.py
```

---

### Step 7: Test Flask App from Jenkins VPC

SSH into the Jenkins EC2 instance and test connectivity to the Flask app over the peered VPC:
```bash
curl http://10.0.2.X:5000
```
(replace `10.0.2.X` with the actual **private IP** of the Flask EC2 instance)

If this returns `Hello from Private Subnet!`, your VPC Peering is working correctly. 🎉

---

### Step 8: Install Jenkins (From Scratch)

**8.1 SSH into the Jenkins EC2 instance:**
```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

**8.2 Install Java (required by Jenkins):**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install openjdk-17-jdk -y
```

**8.3 Add the Jenkins repository and install Jenkins:**
```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins git -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

**8.4 Open port 8080 in the Security Group**

| Type | Protocol | Port Range | Source |
|---|---|---|---|
| Custom TCP | TCP | 8080 | 0.0.0.0/0 (or your IP) |

**8.5 Access Jenkins in the browser:**
```
http://<EC2_PUBLIC_IP>:8080
```

**8.6 Unlock Jenkins:**
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
- Paste the password shown
- Choose **Install Suggested Plugins**
- Create your admin user
- Keep the Jenkins URL as default (or set it to your public IP)

**8.7 Confirm Jenkins is running**

Visit `http://<EC2_PUBLIC_IP>:8080` — you should see the Jenkins dashboard.

---

## 🛠️ Sample Jenkins Pipeline

```groovy
pipeline {
    agent { label 'ahmad' }
    stages {
        stage('Deploy Flask App') {
            steps {
                sh '''
                    pkill -f "python3 /home/ec2-user/app.py" || true
                    sleep 2
                    nohup python3 /home/ec2-user/app.py > /home/ec2-user/flask.log 2>&1 &
                    disown
                '''
            }
        }
    }
}
```

> ⚠️ Don't run `sh "python3 app.py"` directly — Flask's dev server never exits, so Jenkins will hang forever waiting for the build step to finish. Always run it in the background with `nohup ... & disown`, and kill any old instance first to avoid port conflicts.

---

## 🐞 Common Errors & Fixes

### 1. `bash: line 1: python3/home/ec2-user/app.py: command not found`
**Cause:** Missing space in the shell command.
**Fix:**
```groovy
sh "python3 /home/ec2-user/app.py"   // note the space after python3
```

### 2. Jenkins agent fails: `bash: line 1: java: command not found`
**Cause:** Java is not installed on the agent (worker) node. Jenkins agents run on Java — separate from your Python/Flask app.
**Fix:**
```bash
sudo dnf install -y java-17-amazon-corretto   # Amazon Linux 2023
# or
sudo yum install -y java-17-amazon-corretto   # Amazon Linux 2
java -version
```

### 3. `UnsupportedClassVersionError: ... compiled by a more recent version of the Java Runtime`
**Cause:** Version mismatch — the Jenkins **controller** uses a newer Java version than the **agent**. Example: controller runs Java 21, agent only has Java 17.
**Fix:** Install a matching (or newer) Java version on the agent:
```bash
sudo dnf remove -y java-17-amazon-corretto
sudo dnf install -y java-21-amazon-corretto
java -version
sudo alternatives --config java   # select Java 21 if multiple versions exist
```
Then relaunch the agent from **Manage Jenkins → Nodes**.

### 4. Flask app makes the Jenkins build hang forever
**Cause:** `app.run()` blocks and never returns, so the `sh` step waits indefinitely.
**Fix:** Always run it detached:
```bash
nohup python3 app.py > flask.log 2>&1 &
disown
```

### 5. `Address already in use` (port 5000 busy)
**Cause:** An old Flask process is still running from a previous build.
**Fix:** Kill it before starting a new one:
```bash
pkill -f "python3 app.py" || true
```

### 6. Flask app not reachable from Jenkins VPC
**Checklist:**
- Is VPC Peering **active** (not just created)?
- Are routes added on **both** VPC route tables?
- Does the Flask server's Security Group allow inbound **5000** from the Jenkins VPC's CIDR or SG?
- Are you using the **private IP**, not the public IP?

---

## ✅ Conclusion

This project demonstrates a real-world, production-style AWS setup where:
- Workloads are logically separated using **multiple VPCs**
- The application is protected by keeping it in a **private subnet**, reachable only via a **Bastion Host**
- Jenkins securely communicates with the app tier through **VPC Peering**, with no public exposure
- The architecture stays **scalable, modular, and secure** — the same pattern used in enterprise cloud environments

This setup is a solid foundation for anyone learning **DevOps**, **CI/CD**, and **secure AWS networking**.

---

## 📬 Connect

If this helped you, feel free to connect or reach out:

- LinkedIn: [linkedin.com/in/nasirbloch323](https://www.linkedin.com/in/nasirbloch323)
- GitHub: [github.com/nasirbloch323](https://github.com/nasirbloch323)
