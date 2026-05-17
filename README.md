# SOC Home Lab — Splunk SIEM, SOAR Automation & Active Directory Incident Response

A cloud-based Active Directory environment deployed on **Vultr**, integrated with **Splunk** for SIEM-driven threat detection, and **Shuffle** as a SOAR platform to automate incident response workflows. This project simulates a real-world SOC environment, detecting unauthorized logins and automatically remediating compromised accounts via a custom playbook.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Step 1 — Cloud Infrastructure Provisioning (Vultr)](#step-1--cloud-infrastructure-provisioning-vultr)
- [Step 2 — Network Configuration & VPC Setup](#step-2--network-configuration--vpc-setup)
- [Step 3 — Active Directory Setup & Domain Configuration](#step-3--active-directory-setup--domain-configuration)
- [Step 4 — Splunk Deployment, Log Ingestion & Alert Configuration](#step-4--splunk-deployment-log-ingestion--alert-configuration)
- [Step 5 — SOAR Playbook with Shuffle & Slack Integration](#step-5--soar-playbook-with-shuffle--slack-integration)

---

## Project Overview

This project demonstrates end-to-end implementation of a blue team detection and response pipeline using industry-standard tools. The core objectives are:

1. **Build** a functional Active Directory domain environment in the cloud.
2. **Detect** unauthorized or suspicious login activity using Splunk as the SIEM.
3. **Automate** incident response — disabling compromised domain accounts — using Shuffle SOAR with Slack as the analyst notification channel.

**Key Technologies:**

| Tool | Role |
|---|---|
| [Vultr](https://www.vultr.com/) | Cloud infrastructure provider |
| Windows Server 2022 | Domain Controller & endpoint simulation |
| Ubuntu Server 22.04 | Splunk server host |
| Splunk Enterprise | SIEM — log ingestion, correlation, and alerting |
| Splunk Universal Forwarder | Log shipping agent on Windows endpoints |
| Shuffle | SOAR — automated playbook execution |
| Slack | SOC analyst notification & response channel |
| LDAP / Active Directory | Identity management & user remediation target |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Vultr Cloud (VPC)                        │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌───────────────────┐  │
│  │   ADDC01     │    │  TestMac01   │    │   SplunkServer    │  │
│  │ Windows 2022 │    │ Windows 2022 │    │  Ubuntu 22.04     │  │
│  │ Domain Ctrl  │◄──►│ Test Machine │───►│  Splunk :8080     │  │
│  │ 2vCPU/4GB    │    │ 1vCPU/2GB    │    │  4vCPU/8GB/160GB  │  │
│  └──────┬───────┘    └──────┬───────┘    └────────┬──────────┘  │
│         │                   │                     │              │
│         └──────── VPC Private Network ────────────┘              │
└─────────────────────────────────────────────────────────────────┘
                                   │
                          Splunk Alert (Webhook)
                                   │
                    ┌──────────────▼───────────────┐
                    │       Shuffle (SOAR)          │
                    │   Automated Response          │
                    │   Playbook: MyAutoADSpl       │
                    └──────┬───────────────┬────────┘
                           │               │
                    ┌──────▼──────┐ ┌──────▼──────┐
                    │    Slack    │ │  Active Dir  │
                    │   Alerts   │ │ (LDAP :389)  │
                    │  #Alerts   │ │ Disable User │
                    └─────────────┘ └─────────────┘
```

---

## Prerequisites

- A [Vultr](https://www.vultr.com/) account with billing enabled
- A [Splunk](https://www.splunk.com/en_us/download/splunk-enterprise.html) Enterprise free trial license
- A [Shuffle](https://shuffler.io/) account
- A [Slack](https://slack.com/) workspace with admin access
- A local machine with RDP client and SSH capabilities
- Basic familiarity with Windows Server administration and Linux CLI

---

## Step 1 — Cloud Infrastructure Provisioning (Vultr)

Three cloud instances are deployed within the same Vultr region to form the lab environment.

### Instance 1 — Domain Controller (`ADDC01`)

> **Windows Server 2022** acting as the Active Directory Domain Controller. This is the identity backbone of the lab, managing authentication and authorization for all domain-joined machines.

Navigate to **Products → Compute → Deploy New Server** and configure as follows:

| Setting | Value |
|---|---|
| Type | Shared CPU |
| OS | Windows Server 2022 x64 |
| vCPUs | 2 |
| RAM | 4 GB |
| Storage | 50 GB SSD |
| Bandwidth | 3 TB/mo |
| Server Name | `ADDC01` |

### Instance 2 — Test Machine (`TestMac01`)

> **Windows Server 2022** simulating a domain-joined workstation. This endpoint generates authentication events and log telemetry forwarded to Splunk.

| Setting | Value |
|---|---|
| Type | Shared CPU |
| OS | Windows Server 2022 x64 |
| vCPUs | 1 |
| RAM | 2 GB |
| Storage | 50 GB SSD |
| Bandwidth | 2 TB/mo |
| Server Name | `TestMac01` |

### Instance 3 — Splunk Server (`SplunkServer`)

> **Ubuntu Server 22.04** hosting the Splunk Enterprise SIEM instance. Sized appropriately to handle log ingestion, indexing, and search operations from multiple Windows endpoints.

| Setting | Value |
|---|---|
| Type | Shared CPU |
| OS | Ubuntu Server 22.04 |
| vCPUs | 4 |
| RAM | 8 GB |
| Storage | 160 GB SSD |
| Bandwidth | 4 TB/mo |
| Server Name | `SplunkServer` |

### Access Control — Firewall Configuration

> A **Firewall Group** in Vultr functions as a cloud-level security control that filters inbound and outbound traffic to your instances — equivalent to a network ACL or cloud security group.

Navigate to **Products → Network → Firewall → Add Firewall Group** and name it `MyFirewall-ADDC`.

Configure the following inbound rules, restricting access to your IP only:

| Rule | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | Your IP only |
| RDP | TCP (MS RDP) | 3389 | Your IP only |

Apply this firewall group to all three instances via **Settings → Firewall**.

---

## Step 2 — Network Configuration & VPC Setup

### Virtual Private Cloud (VPC)

> A **VPC (Virtual Private Cloud)** is an isolated private network within the cloud provider's infrastructure. Instances within the same VPC can communicate with each other using private IP addresses without traffic leaving the cloud network, improving both security and performance.

Enable VPC on both Windows instances and the Splunk server from the Vultr instance settings. VPC connectivity is scoped to a single region, so all three instances must be deployed in the same geographic region.

Once VPC is enabled, each instance is assigned a private IP address. Configure static private IPs on each Windows machine:

**Start Menu → Network & Internet Settings → Change Adapter Options → Right-click adapter → Properties → IPv4 → Use the following IP address**

Assign each machine its respective VPC-issued static IP and subnet mask. Validate connectivity by pinging the private IPs between instances.

### Accessing Instances

**Domain Controller (`ADDC01`):**
Open **Remote Desktop Connection** on your local machine and connect using the public IP and credentials from the Vultr console.

**Test Machine (`TestMac01`):**
Repeat the same RDP process using `TestMac01`'s public IP.

**Splunk Server (`SplunkServer`):**
Connect via SSH from your terminal:
```bash
ssh username@<SplunkServer_public_ip>
```

Enable VPC and apply the firewall group to the Splunk instance from the Vultr console after connecting.

---

## Step 3 — Active Directory Setup & Domain Configuration

### Promoting `ADDC01` to Domain Controller

RDP into `ADDC01` and open **Server Manager**. Navigate to:

**Add Roles and Features → Server Roles → Active Directory Domain Services → Add Features → Install**

Once installation completes, click **Promote this server to a domain controller** and configure:

| Setting | Value |
|---|---|
| Deployment Operation | Add a new forest |
| Root domain name | `Mylab.local` |
| Directory Services Restore Mode (DSRM) password | *(set a strong password)* |

Complete the wizard and allow the server to restart. On reboot, the instance will be running as a fully configured Active Directory Domain Controller.

### Creating a Domain User

Open **Active Directory Users and Computers**:

**Start → Active Directory Users and Computers → Expand `Mylab.local` → Users → Right-click → New → User**

Create a standard domain user (e.g., `Jane Doe` / username: `JDoe`) with a secure password.

### Joining `TestMac01` to the Domain

RDP into `TestMac01` and navigate to:

**Settings → System → About → Rename this PC (Advanced) → Member of: Domain → Enter `Mylab`**

Provide Domain Admin credentials when prompted and restart the machine.

> **Troubleshooting:** If domain join fails, verify that the **Preferred DNS server** on `TestMac01` is set to the private IP address of `ADDC01`. Active Directory relies on DNS for domain resolution.

After restart, the login screen should display **Sign in to: MYLAB**, confirming successful domain membership. Sign in with `Jane Doe`'s credentials to validate.

### Enabling RDP for Domain User

On `TestMac01`, navigate to:

**Start → Allow remote connections to this computer → Select Users → Add → Enter `JDoe`**

You can now RDP into `TestMac01` using domain credentials:

```
Username: MYLAB\JDoe
Password: <Jane Doe's password>
```

---

## Step 4 — Splunk Deployment, Log Ingestion & Alert Configuration

### Installing Splunk Enterprise on Ubuntu

SSH into the Splunk server and update the system:

```bash
apt-get update && apt-get upgrade -y
```

Download the Splunk Enterprise `.deb` package from the [Splunk website](https://www.splunk.com/en_us/download/splunk-enterprise.html) (free trial — Linux `.deb`). Copy the `wget` download link and run:

```bash
wget -O splunk.deb '<paste_download_link_here>'
dpkg -i splunk.deb
```

Start Splunk and complete the initial configuration:

```bash
cd /opt/splunk/bin
./splunk start
```

Set an admin username and password when prompted. Allow port `8080` through the firewall and Vultr's firewall group (TCP, port `8080`, from your IP only):

```bash
ufw allow 8080
```

Access Splunk via browser at `http://<SplunkServer_public_ip>:8080` and log in with your admin credentials.

### Initial Splunk Configuration

- **Timezone:** User menu → Preferences → Set timezone to `GMT` → Apply
- **Add-on:** Apps → Find More Apps → Install **Splunk Add-on for Microsoft Windows**
- **Index:** Settings → Indexes → New Index → Name: `MyProjAD` → Save
- **Receiver Port:** Settings → Forwarding and Receiving → Configure Receiving → New Receiving Port → `9997`

Allow the forwarder port through the host firewall:

```bash
ufw allow 9997
```

### Installing Splunk Universal Forwarder on Windows Endpoints

> The **Splunk Universal Forwarder** is a lightweight agent installed on endpoint machines that collects and ships log data (events, security logs, system logs) to a centralized Splunk indexer without requiring a full Splunk installation on the endpoint.

Download the Universal Forwarder from the [Splunk website](https://www.splunk.com/en_us/download/universal-forwarder.html) and install on both `ADDC01` and `TestMac01`.

During installation, configure the **Receiving Indexer** as:

| Setting | Value |
|---|---|
| Indexer Host | `<SplunkServer VPC private IP>` |
| Indexer Port | `9997` |

### Configuring `inputs.conf`

> `inputs.conf` is the Splunk Universal Forwarder configuration file that defines which data sources (event logs, files, scripts) are monitored and forwarded to the indexer.

Navigate to `C:\Program Files\SplunkUniversalForwarder\etc\system\local\` and open or create `inputs.conf` as Administrator. Add the following stanza:

```ini
[WinEventLog://Security]
index = MyProjAD
disabled = False
```

Restart the Splunk Forwarder service to apply changes:

**Services (run as admin) → SplunkForwarder → Log On → Local System Account → Apply → Restart**

Validate log ingestion in Splunk by running the following search:

```
index=MyProjAD
```

Repeat the Universal Forwarder installation and `inputs.conf` configuration on the Domain Controller (`ADDC01`).

### Simulating Brute-Force / Unauthorized Login Activity

To simulate real-world attack traffic, temporarily open RDP to the public internet:

**Vultr Firewall → Modify MS RDP rule → Allow TCP port `3389` from Anywhere**

This exposes the endpoints to external login attempts, generating the `EventCode 4624` (successful logon) events used by the Splunk detection query.

### Creating a Splunk Detection Alert

> **EventCode 4624** is a Windows Security Event Log entry that records a successful account logon. **Logon Type 7** indicates an unlock event (interactive console), while **Logon Type 10** indicates a Remote Interactive logon (RDP). Monitoring these events from external source IPs is a key indicator of unauthorized access.

Navigate to **Apps → Search and Reporting** and run the following detection query:

```spl
index=MyProjAD EventCode=4624 (Logon_Type=7 OR Logon_Type=10) 
Source_Network_Address=* Source_Network_Address=40.*
| stats count by _time, ComputerName, Source_Network_Address, user, Logon_Type
```

**Query breakdown:**

| Clause | Purpose |
|---|---|
| `index=MyProjAD` | Scopes the search to the lab's dedicated Splunk index |
| `EventCode=4624` | Filters for successful Windows logon events only |
| `Logon_Type=7 OR Logon_Type=10` | Targets interactive (unlock) and RDP logon types |
| `Source_Network_Address=40.*` | Filters for external source IPs (simulating public internet traffic) |
| `stats count by ...` | Aggregates events by timestamp, hostname, source IP, user, and logon type |

Save the search as an Alert and configure as follows:

| Setting | Value |
|---|---|
| Alert Name | `ADDC01-Unauthorized_Success_Login_Alert` |
| Schedule | Cron: `* * * * *` (every minute) |
| Trigger Condition | Per result |
| Trigger Action | Add to Triggered Alerts (severity: High) |

Monitor triggered alerts at **Activity → Triggered Alerts**.

---

## Step 5 — SOAR Playbook with Shuffle & Slack Integration

### Overview

> **SOAR (Security Orchestration, Automation, and Response)** platforms enable security teams to automate repetitive response tasks by building workflows that integrate multiple security tools. **Shuffle** is an open-source SOAR platform that connects Splunk alerts to response actions — in this case, automatically disabling a compromised domain user account via LDAP.

The playbook (`MyAutoADSpl`) executes the following logic upon alert trigger:

```
Splunk Alert ──► Shuffle Webhook ──► Slack Notification
                                          │
                                   SOC Analyst Review
                                          │
                              [Disable User? Yes / No]
                                          │
                              Active Directory (LDAP)
                              Disable Domain Account
                                          │
                              Verify Account Status
                                          │
                              Slack Confirmation Message
```

### Workflow Setup in Shuffle

1. Log in to [Shuffle](https://shuffler.io/) and navigate to **Workflows → Create Workflow**.
2. Name the workflow `MyAutoADSpl` and save.

### Step 5a — Splunk Webhook Trigger

Inside the workflow, add a **Webhook** app:

- Name: `Splunk-Alert`
- Copy the generated webhook URL.

In Splunk, navigate to **Apps → Search and Reporting → Alerts → ADDC01-Unauthorized_Success_Login_Alert → Edit Alert → Trigger Actions → Add Action → Webhook** → paste the Shuffle webhook URL → Save.

### Step 5b — Slack Notification

Create a dedicated Slack workspace and a private channel named `#alerts` for receiving SOC notifications.

In the Shuffle workflow, add the **Slack** app:

- Authenticate with your Slack account via OAuth.
- Name the app node: `SlackAlertNotification`.
- Configure the message **Text** field to include dynamic values from the Splunk alert payload:

```
🚨 Unauthorized Login Detected
Alert: ADDC01-Unauthorized_Success_Login_Alert
Time: $exec.result._time
User: $exec.result.user
Source IP: $exec.result.Source_Network_Address
```

- Set the **Channel** field to your `#alerts` channel ID (obtainable from the Slack URL).
- Connect the **Splunk Webhook** node to the **SlackAlertNotification** node in the workflow canvas.

### Step 5c — Analyst Decision (User Input Trigger)

Add a **User Input** trigger to introduce a human-in-the-loop approval step:

- Name: `User_Action`
- Prompt: `Would you like to disable this user?`
- Input method: **Email** → enter your analyst email address.
- Connect `User_Action` to the `SlackAlertNotification` node.

### Step 5d — Active Directory Integration (LDAP)

> **LDAP (Lightweight Directory Access Protocol)** is the protocol used to communicate with Active Directory for querying and modifying directory objects, such as enabling or disabling user accounts.

Allow LDAP traffic from Shuffle's IP through Vultr's firewall:

**Vultr Firewall → Add rule → TCP → Port `389` → From Anywhere → Save**

Add the **Active Directory** app to the workflow and authenticate:

| Field | Value |
|---|---|
| Server | `<ADDC01 public IP>` |
| Port | `389` |
| Domain | `Mylab` |
| Username | Domain Admin |
| Password | Domain Admin password |
| Base DN | *(retrieve with `Get-ADDomain` in PowerShell → copy `UsersContainer` value)* |
| Use SSL | `False` |

Configure the Active Directory app node:

- **Action:** `Disable User`
- **SamAccountName:** `$exec.result.user`
- **Search Base:** `<UsersContainer DN>`

### Step 5e — Account Status Verification

Add a second **Active Directory** app node named `Get_User_Attributes` and connect it to the first AD node. This fetches the user's current attributes post-remediation to confirm the account has been disabled.

Add a **Shuffle Tools** (Repeat Back to Me) app node named `Check_AD_User`:

- **Action:** `Repeat back to me`
- **Call:** `Get_User_Attributes → UserAccountControl`

Connect `Check_AD_User` to the `Get_User_Attributes` node.

### Step 5f — Remediation Confirmation in Slack

Add a final **Slack** app node to send confirmation of the account disable action:

- **Text:** `✅ Account $exec.result.user has been DISABLED.`
- **Channel:** `<#alerts channel ID>`

Connect this node to `Check_AD_User` with the following **line condition**:

```
if $get_user_attributes.attributes.userAccountControl Contains "ACCOUNTDISABLED"
```

This ensures the confirmation message is only sent after verifying the account is genuinely disabled in Active Directory.

---

## Workflow Summary

| Step | Component | Action |
|---|---|---|
| 1 | Vultr | Provision 3 cloud instances (DC, Test Machine, Splunk) |
| 2 | Vultr VPC + Firewall | Isolate instances on private network, restrict public access |
| 3 | Windows Server / AD | Promote DC, create domain, join endpoint, configure user |
| 4 | Splunk + UF | Ingest Windows Security logs, build detection query, create alert |
| 5 | Shuffle + Slack + AD | Automate analyst notification and user account remediation |

---

## Security Considerations

- Restrict firewall rules to your IP wherever possible. The public RDP exposure in Step 4 is intentional for simulation purposes only — revert this rule after testing.
- Use strong, unique passwords for all domain accounts, Splunk, and Shuffle authentication.
- LDAP in this lab runs over port 389 (unencrypted). In production environments, use **LDAPS (port 636)** with a valid SSL certificate.
- Rotate all credentials upon completion of the lab.

---

## References

- [Splunk Documentation](https://docs.splunk.com/)
- [Shuffle SOAR Documentation](https://shuffler.io/docs)
- [Microsoft Active Directory DS Overview](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview)
- [Vultr Documentation](https://docs.vultr.com/)
- [Windows Security Event ID Reference](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/)
