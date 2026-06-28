# Setup Guide — Endpoint Security & Monitoring with Wazuh

## Prerequisites

| Requirement | Spec |
|-------------|------|
| Wazuh Server OS | Ubuntu Server 22.04 LTS |
| RAM | 4GB minimum, 8GB recommended |
| Disk | 50GB minimum |
| Endpoints | Windows 10/11 or Linux |
| Network | All endpoints must reach Wazuh Manager on port 1514/1515 |

---

## Step 1: Deploy Wazuh Manager (Ubuntu Server)

### 1.1 Update system
```bash
sudo apt-get update && sudo apt-get upgrade -y
```

### 1.2 Run Wazuh all-in-one installer
```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a
```
This installs: Wazuh Manager + Indexer + Dashboard (~10-15 minutes).

At the end, note down the admin credentials printed in the terminal.

### 1.3 Verify services are running
```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

### 1.4 Access the Dashboard
Open a browser and navigate to:
```
https://<your-server-ip>
```
Login with the `admin` credentials from the installer output.

---

## Step 2: Deploy Sysmon on Windows Endpoint

### 2.1 Download Sysmon
Download from Microsoft Sysinternals:
```
https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
```

### 2.2 Install with our custom config
Copy `configs/sysmon-config.xml` to the same folder as Sysmon, then run in an **admin PowerShell**:
```powershell
.\Sysmon64.exe -accepteula -i sysmon-config.xml
```

### 2.3 Verify Sysmon is running
```powershell
Get-Service sysmon64
```
Should show: `Running`

### 2.4 Enable PowerShell Script Block Logging
```powershell
# Run in admin PowerShell
$basePath = "HKLM:\Software\Policies\Microsoft\Windows\PowerShell"
New-Item $basePath\ScriptBlockLogging -Force
Set-ItemProperty $basePath\ScriptBlockLogging EnableScriptBlockLogging 1
```

---

## Step 3: Install Wazuh Agent on Windows

### 3.1 Download agent installer
```
https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.0-1.msi
```

### 3.2 Install via PowerShell (admin)
```powershell
msiexec /i wazuh-agent-4.7.0-1.msi /q WAZUH_MANAGER="<YOUR_MANAGER_IP>"
```

### 3.3 Apply our custom agent config
Copy `configs/wazuh-agent.conf` to:
```
C:\Program Files (x86)\ossec-agent\ossec.conf
```
(Replace the existing file — edit `WAZUH_MANAGER_IP` with your actual IP first)

### 3.4 Start the agent
```powershell
NET START WazuhSvc
```

### 3.5 Verify agent appears in dashboard
Go to Wazuh Dashboard → Agents → the new Windows agent should appear as **Active**.

---

## Step 4: Apply Custom Detection Rules

### 4.1 Copy rules to Wazuh Manager
```bash
sudo cp rules/custom-rules.xml /var/ossec/etc/rules/custom-rules.xml
```

### 4.2 Validate rules syntax
```bash
sudo /var/ossec/bin/wazuh-logtest -V
```

### 4.3 Reload rules
```bash
sudo /var/ossec/bin/wazuh-control reload
```

---

## Step 5: Set Up MalwareBazaar Threat Intel Feed

### 5.1 Install Python dependencies
```bash
sudo apt install python3-pip -y
pip3 install requests
```

### 5.2 Copy script to server
```bash
sudo cp scripts/malwarebazaar-feed.py /opt/scripts/
sudo chmod +x /opt/scripts/malwarebazaar-feed.py
```

### 5.3 Run manually to test
```bash
sudo python3 /opt/scripts/malwarebazaar-feed.py
```

### 5.4 Set up cron job (runs every 6 hours)
```bash
sudo crontab -e
```
Add this line:
```
0 */6 * * * /usr/bin/python3 /opt/scripts/malwarebazaar-feed.py >> /var/log/malwarebazaar-feed.log 2>&1
```

---

## Step 6: Configure Alerting

### 6.1 Email alerts (in `/var/ossec/etc/ossec.conf`)
```xml
<global>
  <email_notification>yes</email_notification>
  <email_to>security@internee.pk</email_to>
  <smtp_server>smtp.gmail.com</smtp_server>
  <email_from>wazuh@internee.pk</email_from>
</global>

<alerts>
  <email_alert_level>10</email_alert_level>
</alerts>
```

### 6.2 Webhook alerts (Slack/Discord)
Edit `scripts/alert-webhook.py` with your webhook URL, then:
```bash
sudo cp scripts/alert-webhook.py /var/ossec/integrations/
sudo chmod 750 /var/ossec/integrations/alert-webhook.py
```

Add to `/var/ossec/etc/ossec.conf`:
```xml
<integration>
  <name>alert-webhook</name>
  <level>12</level>
  <alert_format>json</alert_format>
</integration>
```

---

## Step 7: Verify Everything Works

### 7.1 Simulate a brute force attack (safe test)
On the Windows endpoint, run this in PowerShell to generate failed logins:
```powershell
# This generates failed login events safely
1..10 | ForEach-Object { 
  $cred = New-Object System.Management.Automation.PSCredential("fakeuser", (ConvertTo-SecureString "wrongpass" -AsPlainText -Force))
  Start-Process cmd -Credential $cred -ErrorAction SilentlyContinue 2>$null
}
```

### 7.2 Check Wazuh Dashboard
Go to: **Security Events** → filter by `rule.id: 100002`
You should see the brute force alert fire within 60 seconds.

### 7.3 Test FIM
Create a file in `C:\Windows\System32\test.txt` (as admin) — FIM should alert within 5 minutes.

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Agent shows as disconnected | Check firewall allows port 1514 TCP |
| No Sysmon events in Wazuh | Verify `Microsoft-Windows-Sysmon/Operational` in agent config |
| Rules not triggering | Run `wazuh-logtest` to validate rule syntax |
| MalwareBazaar script fails | Check internet connectivity from Wazuh server |
