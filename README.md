# 🛡️ Wazuh on Ubuntu — Lab Manual

> **Connect via PuTTY | Install Wazuh Manager & Agent | Explore the Dashboard**

---

## 📋 Table of Contents

1. [Lab Overview](#lab-overview)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Connect to Ubuntu Server via PuTTY](#step-1--connect-to-ubuntu-server-via-putty)
4. [Step 2 — Prepare the Ubuntu Server](#step-2--prepare-the-ubuntu-server)
5. [Step 3 — Install Wazuh Manager (Single-Node)](#step-3--install-wazuh-manager-single-node)
6. [Step 4 — Access the Wazuh Dashboard](#step-4--access-the-wazuh-dashboard)
7. [Step 5 — Install Wazuh Agent on Ubuntu](#step-5--install-wazuh-agent-on-ubuntu)
8. [Step 6 — Verify Agent Connection](#step-6--verify-agent-connection)
9. [Troubleshooting](#troubleshooting)
10. [Lab Summary](#lab-summary)

---

## Lab Overview

| Detail | Value |
|--------|-------|
| **Goal** | Deploy Wazuh SIEM on Ubuntu and connect using PuTTY |
| **Wazuh Version** | 4.x (latest stable) |
| **OS** | Ubuntu 22.04 LTS (server & agent) |
| **Access Tool** | PuTTY (SSH client for Windows) |
| **Estimated Time** | 60–90 minutes |

**Wazuh** is an open-source security platform that provides:
- 🔍 Intrusion Detection (HIDS)
- 📊 Log Data Analysis
- 🔒 File Integrity Monitoring (FIM)
- 🌐 Security Dashboard (OpenSearch-based)

---

## Prerequisites

### Software Required

| Tool | Download Link | Purpose |
|------|--------------|---------|
| **PuTTY** | https://www.putty.org | SSH client (Windows) |
| **Ubuntu 22.04 LTS** | https://ubuntu.com/download/server | Server OS |
| **VirtualBox / VMware** | https://www.virtualbox.org | VM platform (if no bare-metal) |

### System Requirements (Wazuh Server)

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 2 cores | 4 cores |
| RAM | 4 GB | 8 GB |
| Disk | 50 GB | 100 GB |
| OS | Ubuntu 20.04+ | Ubuntu 22.04 LTS |

### Network Setup

- Ubuntu Server must have a **static IP** or know its IP address
- PuTTY machine must be able to **reach that IP** (same network or NAT with port forwarding)
- SSH port **22** must be open on the Ubuntu server

---

## Step 1 — Connect to Ubuntu Server via PuTTY

### 1.1 Find the Ubuntu Server IP

On the Ubuntu server console, run:

```bash
ip a
```

Look for `inet` under your network interface (e.g., `ens33`, `eth0`):

```
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.1.100/24
```

> 📝 **Note your IP address** — you will need it in PuTTY.

---

### 1.2 Enable SSH on Ubuntu (if not already enabled)

```bash
sudo apt update
sudo apt install openssh-server -y
sudo systemctl enable ssh
sudo systemctl start ssh
sudo systemctl status ssh
```

Expected output:
```
● ssh.service - OpenBSD Secure Shell server
   Active: active (running)
```

---

### 1.3 Configure PuTTY

1. **Open PuTTY** on your Windows machine
2. In the **Host Name (or IP address)** field, enter your Ubuntu IP (e.g., `192.168.1.100`)
3. Set **Port** to `22`
4. Connection type: **SSH**
5. *(Optional)* Under **Connection > Data**, set your **Auto-login username**
6. Click **Open**

```
┌─────────────────────────────────────┐
│  PuTTY Configuration                │
│                                     │
│  Host Name: 192.168.1.100           │
│  Port:      22                      │
│  Connection type: ● SSH             │
│                                     │
│           [ Open ]                  │
└─────────────────────────────────────┘
```

7. Accept the **host key fingerprint** (click Yes on first connection)
8. Enter your **Ubuntu username** and **password**

> ✅ You should now be logged into your Ubuntu server remotely via PuTTY.

---

## Step 2 — Prepare the Ubuntu Server

### 2.1 Update the System

```bash
sudo apt update && sudo apt upgrade -y
```

### 2.2 Set the Hostname (Optional but Recommended)

```bash
sudo hostnamectl set-hostname wazuh-server
```

Verify:
```bash
hostnamectl
```

### 2.3 Disable the Firewall Temporarily (Lab Only)

> ⚠️ For production, configure UFW rules instead of disabling.

```bash
sudo ufw disable
```

Or allow required ports:
```bash
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 443/tcp     # Wazuh Dashboard (HTTPS)
sudo ufw allow 1514/tcp    # Wazuh agent communication
sudo ufw allow 1515/tcp    # Wazuh agent enrollment
sudo ufw allow 9200/tcp    # OpenSearch
sudo ufw enable
```

---

## Step 3 — Install Wazuh Manager (Single-Node)

Wazuh provides an **automated installation script** that sets up:
- Wazuh Manager
- OpenSearch (Indexer)
- Wazuh Dashboard

### 3.1 Download the Installation Script

```bash
curl -sO https://packages.wazuh.com/4.8/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.8/config.yml
```

### 3.2 Edit the Configuration File

```bash
nano config.yml
```

Replace the default IP with your server's actual IP:

```yaml
nodes:
  # Wazuh indexer nodes
  indexer:
    - name: node-1
      ip: "192.168.1.100"     # ← Change this to your server IP

  # Wazuh server nodes
  server:
    - name: wazuh-1
      ip: "192.168.1.100"     # ← Change this to your server IP

  # Wazuh dashboard nodes
  dashboard:
    - name: dashboard
      ip: "192.168.1.100"     # ← Change this to your server IP
```

Save and exit: `Ctrl+O`, `Enter`, `Ctrl+X`

### 3.3 Generate Configuration Files

```bash
bash wazuh-install.sh --generate-config-files
```

### 3.4 Install the Wazuh Indexer

```bash
bash wazuh-install.sh --wazuh-indexer node-1
```

> ⏳ This may take 5–10 minutes.

### 3.5 Start the Wazuh Cluster

```bash
bash wazuh-install.sh --start-cluster
```

### 3.6 Install the Wazuh Server

```bash
bash wazuh-install.sh --wazuh-server wazuh-1
```

### 3.7 Install the Wazuh Dashboard

```bash
bash wazuh-install.sh --wazuh-dashboard dashboard
```

### 3.8 Retrieve the Admin Password

```bash
tar -axf wazuh-install-files.tar.gz wazuh-install-files/wazuh-passwords.txt -O | grep -P "\'admin\'" -A 1
```

> 📝 **Save this password** — you will use it to log into the Dashboard.

Example output:
```
  username: "admin"
  password: "Ab12Cd34Ef56Gh78"
```

---

## Step 4 — Access the Wazuh Dashboard

### 4.1 Open the Dashboard in a Browser

On your Windows machine, open a browser and navigate to:

```
https://192.168.1.100
```

> ⚠️ You will see a **SSL certificate warning** — this is expected in a lab. Click **Advanced → Proceed**.

### 4.2 Log In

| Field | Value |
|-------|-------|
| **Username** | `admin` |
| **Password** | *(from Step 3.8)* |

### 4.3 Dashboard Overview

After login, you will see:

```
┌────────────────────────────────────────────────────┐
│  🛡️  Wazuh Dashboard                               │
│                                                    │
│  Agents: 0 active    Security Events: 0            │
│                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │ Threat   │  │ File     │  │ Vuln.    │         │
│  │ Detection│  │ Integrity│  │ Detector │         │
│  └──────────┘  └──────────┘  └──────────┘         │
└────────────────────────────────────────────────────┘
```

---

## Step 5 — Install Wazuh Agent on Ubuntu

The **Wazuh Agent** can be installed on any endpoint you want to monitor. In this lab, we install it on the **same Ubuntu server** (or a second Ubuntu VM).

### 5.1 Add the Wazuh Repository

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | \
  gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && \
  chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt update
```

### 5.2 Install the Wazuh Agent

```bash
WAZUH_MANAGER="192.168.1.100" sudo apt install wazuh-agent -y
```

> 🔁 Replace `192.168.1.100` with your **Wazuh Manager IP**.

### 5.3 Start and Enable the Agent

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

### 5.4 Check Agent Status

```bash
sudo systemctl status wazuh-agent
```

Expected:
```
● wazuh-agent.service - Wazuh agent
   Active: active (running)
```

---

## Step 6 — Verify Agent Connection

### 6.1 On the Wazuh Manager

Check connected agents:

```bash
sudo /var/ossec/bin/agent_control -la
```

Expected output:
```
ID: 001, Name: ubuntu-agent, IP: 192.168.1.100, Active
```

### 6.2 On the Dashboard

1. Go to **Wazuh Dashboard** → `https://192.168.1.100`
2. Navigate to **Agents** tab
3. You should see your agent listed as **Active** ✅

```
┌────────────────────────────────────────────┐
│  Agents                                    │
│                                            │
│  001  ubuntu-agent  192.168.1.100  Active  │
│                                            │
└────────────────────────────────────────────┘
```

### 6.3 Explore Security Events

1. Click on the agent name
2. Navigate to **Security Events**
3. You will see real-time logs from the endpoint

---

## Troubleshooting

### ❌ PuTTY: "Connection refused"

```bash
# On Ubuntu server:
sudo systemctl status ssh
sudo systemctl start ssh
# Also check the IP is correct:
ip a
```

### ❌ Dashboard not loading

```bash
sudo systemctl status wazuh-dashboard
sudo systemctl restart wazuh-dashboard
# Check logs:
sudo journalctl -u wazuh-dashboard -n 50
```

### ❌ Wazuh Manager not running

```bash
sudo systemctl status wazuh-manager
sudo systemctl restart wazuh-manager
# Check manager logs:
sudo tail -f /var/ossec/logs/ossec.log
```

### ❌ Agent shows "Disconnected"

```bash
# On the agent machine:
sudo systemctl restart wazuh-agent
# Check agent logs:
sudo tail -f /var/ossec/logs/ossec.log
# Verify manager IP in config:
sudo cat /var/ossec/etc/ossec.conf | grep server-ip
```

### ❌ Wrong password for Dashboard

```bash
# Extract all passwords again:
tar -axf wazuh-install-files.tar.gz wazuh-install-files/wazuh-passwords.txt -O
```

---

## Lab Summary

| Step | Task | Status |
|------|------|--------|
| 1 | Connect to Ubuntu via PuTTY | ✅ |
| 2 | Prepare Ubuntu (update, SSH, firewall) | ✅ |
| 3 | Install Wazuh Manager (all-in-one) | ✅ |
| 4 | Access Wazuh Dashboard via browser | ✅ |
| 5 | Install and register Wazuh Agent | ✅ |
| 6 | Verify agent in Dashboard | ✅ |

### 🎓 Key Concepts Covered

- **SSH remote access** with PuTTY
- **Wazuh Single-Node deployment** (Manager + Indexer + Dashboard)
- **Agent enrollment** and registration
- **SIEM Dashboard** navigation
- **Log monitoring** and security event analysis

---

## 📚 References

- [Wazuh Official Documentation](https://documentation.wazuh.com)
- [Wazuh Quickstart Guide](https://documentation.wazuh.com/current/quickstart.html)
- [PuTTY Documentation](https://www.chiark.greenend.org.uk/~sgtatham/putty/docs.html)
- [Ubuntu 22.04 Server Guide](https://ubuntu.com/server/docs)

---

> 📁 **Repository Structure (suggested)**
> ```
> wazuh-lab/
> ├── README.md          ← This file
> ├── config.yml         ← Wazuh node config template
> └── screenshots/       ← Add your lab screenshots here
> ```

---

*Lab manual created for educational purposes. Tested on Ubuntu 22.04 LTS with Wazuh 4.8.*
