# Setting Up a Wazuh Honeypot on Azure Cloud: A Step-by-Step Guide

## Introduction

A honeypot is a decoy system designed to attract attackers, enabling the monitoring of malicious activity and gaining insights into attack vectors. In this guide, we will deploy a honeypot in Microsoft Azure using Wazuh, an open-source security platform, as the monitoring agent. This document provides detailed steps and references to set up the environment.

---

## Learning Objectives

- Deploy a virtual machine (VM) on Azure.
- Install and configure the Wazuh agent.
- Set up honeypot services to attract attackers.
- Monitor and analyze logs using Wazuh.

---

## Prerequisites

1. A Microsoft Azure account with sufficient credits.
2. Basic knowledge of Linux and networking.
3. SSH client (e.g., PuTTY or OpenSSH).

---

## Step 1: Create an Azure Virtual Machine

### 1.1 Create an Azure Account

- Sign up for an Azure account at [Azure Portal](https://portal.azure.com).
- New users receive $200 in free credits.

### 1.2 Deploy a Virtual Machine

1. Log in to the Azure portal.
2. Search for "Virtual Machines" and click "Create".
3. Fill in the following details:
   - **Resource Group**: Create a new group (e.g., `honeypot-rg`).
   - **VM Name**: Name your VM (e.g., `honeypot-vm`).
   - **Region**: Choose your preferred region.
   - **Image**: Select Ubuntu Server 22.04 LTS or Debian.
   - **Size**: Choose a size suitable for your workload (e.g., Standard B1s).
4. Set up an admin username and password or SSH key.
5. Under "Inbound Port Rules," allow SSH (port 22) and other ports required for honeypot services.
6. Click "Review + Create," then "Create."

> Reference: [Azure VM Setup Guide](https://www.linkedin.com/pulse/creating-honeypot-microsoft-azure-tutorial-gustavo-gradilla-vcv7c)

---

## Step 2: Install and Configure the Wazuh Agent

### 2.1 Connect to the VM

Use an SSH client to connect to your VM:

```bash
ssh <username>@<vm_public_ip>
```
### 2.2 Install Wazuh Agent

```
curl -sO https://packages.wazuh.com/key/GPG-KEY-WAZUH
sudo apt-key add GPG-KEY-WAZUH
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
```
#### Install the Wazuh agent:
```
sudo apt update && sudo apt install wazuh-agent -y
```
#### Configure the agent to connect to your Wazuh server:
```
sudo nano /var/ossec/etc/ossec.conf
```
#### Start and enable the agent:
```
sudo systemctl start wazuh-agent
sudo systemctl enable wazuh-agent

</https://documentation.wazuh.com/current/quickstart.html>
```

### Step 3: Set Up Honeypot Services
### 3.1 Install Cowrie SSH Honeypot - Cowrie is a popular low-interaction honeypot that simulates an SSH server.
```
Update the system: sudo apt update && sudo apt upgrade -y
Install dependencies: sudo apt install git python3 python3-virtualenv libssl-dev libffi-dev build-essential -y
Clone the Cowrie repository: git clone https://github.com/cowrie/cowrie.git /opt/cowrie
Set up Cowrie:

cd /opt/cowrie
virtualenv cowrie-env
source cowrie-env/bin/activate
pip install --upgrade pip twisted pyasn1
cp cowrie.cfg.dist cowrie.cfg
./bin/cowrie start

```
### Step 4: Configure Network Security Rules

1. Modify inbound rules in Azure to expose ports for honeypot services:

2. Go to your VM's "Networking" tab in the Azure portal.

3. Add rules for ports used by Cowrie (e.g., port 22 or custom ports).

Example rule:
- Source: Any
- Destination Port Ranges: *
- Protocol: TCP

### Step 5: Monitor Logs Using Wazuh

#### 5.1 Access Wazuh Dashboard
- Open your browser and navigate to your Wazuh server's web interface.

#### 5.2 Analyze Security Events
- Navigate to "Discover" or "Security Events."
- Filter logs by the honeypot VM's IP address.
- Look for patterns such as brute-force attempts or unusual activity.


### Step 6: Clean Up Resources
- az group delete --name <resource_group_name> --yes --no-wait



## References

- [Creating a Honeypot with Microsoft Azure](https://learn.microsoft.com/en-us/azure/)
- [Wazuh Quickstart Documentation](https://documentation.wazuh.com/current/installation-guide/index.html)
- [Cowrie GitHub Repository](https://github.com/cowrie/cowrie)
- [T-POT Honeypot Setup on Azure](https://github.com/telekom-security/tpotce)
- [YouTube Tutorial on Wazuh Honeypots](https://www.youtube.com/results?search_query=wazuh+honeypot)














