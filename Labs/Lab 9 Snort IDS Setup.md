# Snort IDS Lab 

### Installation → Detection → Blocking → Email Alerts → Testing with Kali

---

## Overview

This documentation walks through a complete Snort Intrusion Detection System (IDS) lab setup using:

- **Ubuntu (Snort Server)** – Monitoring and protecting the system

- **Kali Linux (Attacker Machine)** – Generating ping and Nmap scans

- **Snort** – Detecting suspicious activity

- **UFW Firewall** – Blocking attackers automatically

- **Email Notifications** – Sending alert emails when attacks occur


By the end of this guide, your Ubuntu machine will:

- Detect ICMP pings

- Detect Nmap port scans

- Automatically block the attacker

- Send you email alerts


---

## 1. Lab Environment Setup

## VirtualBox Network Configuration

Ensure both Ubuntu and Kali are connected to the **same network**:

Recommended:

- **Host-Only Adapter**  

- **Internal Network**


Verify Ubuntu IP address:

```
ip addr
```

Example:

```
192.168.56.101
```

---

## 2. Installing Snort on Ubuntu

#### Step 1: Update System

```
sudo apt update && sudo apt upgrade -y
```

#### Step 2: Install Dependencies

```
sudo apt install -y build-essential libpcap-dev libpcre3-dev \
libdumbnet-dev zlib1g-dev liblzma-dev openssl libssl-dev
```

##### Step 3: Install Snort

```
sudo apt install -y snort
```

During installation:

- Select your **network interface** (example: `enp0s3`)

- Enter your **home network range** (example: `192.168.56.0/24`)


---

##### Step 4: Verify Installation

```
snort -V
```

If version information appears, installation was successful.

To make UFW start automatically at boot on machine, run the following command:

```
sudo systemctl enable ufw
```

---

#### ⚙️ 3. Configuring Snort

Open configuration file:

```
sudo nano /etc/snort/snort.conf
```

##### Set HOME_NET

Find:

```
sudo cat /etc/snort/snort.conf | grep "ipvar HOME_NET"
```


```
ipvar HOME_NET any
```

Change to:

```
ipvar HOME_NET 192.168.56.0/24
```

##### Ensure local rules are included


Add this line if not present:

```
sudo cat /etc/snort/snort.conf | grep "$RULE_PATH/local.rules"
```

```
include $RULE_PATH/local.rules
```

---

## 📜 4. Creating Custom Detection Rules

Create or edit:

```
sudo nano /etc/snort/rules/local.rules
```

Add:

```snort
# ============================================================

# LOCAL SNORT RULES

# Detect Ping and Nmap Scans

# ============================================================

# -----------------------------

# [1] ICMP Ping Detection

# -----------------------------

alert icmp any any -> $HOME_NET any (msg:"ICMP Ping Detected"; itype:8; sid:1000001; rev:1;)

# -----------------------------

# [2] Nmap SYN Scan Detection

# -----------------------------

alert tcp any any -> $HOME_NET any (flags:S; msg:"Nmap SYN Scan Detected"; threshold:type both, track by_src, count 20, seconds 10; sid:1000002; rev:1;)

# -----------------------------

# [3] Nmap Connect Scan Detection

# -----------------------------

alert tcp any any -> $HOME_NET any (flags:S; msg:"Nmap Connect Scan Detected"; threshold:type both, track by_src, count 10, seconds 5; sid:1000003; rev:1;)

# -----------------------------

# [4] Nmap Xmas Scan Detection

# -----------------------------

alert tcp any any -> $HOME_NET any (flags:FPU; msg:"Nmap Xmas Scan Detected"; sid:1000004; rev:1;)

# -----------------------------

# [5] Nmap NULL Scan Detection

# -----------------------------

alert tcp any any -> $HOME_NET any (flags:0; msg:"Nmap NULL Scan Detected"; sid:1000005; rev:1;)

# -----------------------------

# [6] Nmap FIN Scan Detection

# -----------------------------

alert tcp any any -> $HOME_NET any (flags:F; msg:"Nmap FIN Scan Detected"; sid:1000006; rev:1;)

```

Save and exit.

```
sudo systemctl restart snort
sudo systemctl status snort
```

---

#### Test Configuration

```
sudo snort -T -c /etc/snort/snort.conf
```

If successful, you’ll see:

```
Snort successfully validated the configuration
```

---

## 5. Running Snort

Run Snort in alert logging mode. To make it log alerts to a file, run it like this:

```
sudo snort -A fast -c /etc/snort/snort.conf -i enp0s3
```

Run in console so you can watch alerts immediately:

```
sudo snort -A console -q -c /etc/snort/snort.conf -i enp0s3
```

Alerts will be written to:

```
/var/log/snort/alert
```

Monitor alerts live:

```
sudo tail -f /var/log/snort/alert
```

---

## 6. Testing Detection from Kali

On Kali Linux:

**Ping Test**

```
ping -c 4 192.168.56.101
```

**Nmap Scan**

```
sudo nmap -sS 192.168.56.101
```

Snort should generate alerts.

---

## 7. Automatically Blocking the Attacker (UFW Integration)

**Install UFW**

```
sudo apt install ufw -y
sudo ufw enable
```

---

**Create Auto-Block Script**

```
sudo nano /usr/local/bin/snort-block.sh
```

Add:

```bash
#!/bin/bash
LOGFILE="/var/log/snort/alert"
BLOCKED="/var/log/snort/blocked_ips.txt"

tail -Fn0 $LOGFILE | while read line; do
    IP=$(echo $line | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' | head -1)
    if [ ! -z "$IP" ]; then
        if ! grep -q "$IP" $BLOCKED 2>/dev/null; then
            ufw deny from $IP
            echo $IP >> $BLOCKED
        fi
    fi
done
```

Make executable:

```
sudo chmod +x /usr/local/bin/snort-block.sh
```

Restart Snort:

```
sudo systemctl restart snort
sudo systemctl status snort
```

Run the ip block script:

```
sudo bash /usr/local/bin/snort-block.sh
```

Now Kali will be automatically blocked after triggering an alert.

---

## 8. Sending Snort Alerts to Email

Next step is to make snort send the alerts to our Gmail Inbox.

#### Gmail requires an **App Password**

Google **does not allow normal passwords** for SMTP anymore.

You must create one:

1. Go to **Google Account Security**

2. Enable **2-Step Verification**

3. Open **App Passwords**

4. Generate password for **Mail**

5. Copy the **16-character password**

Example:

```
abcd efgh ijkl mnop
```

Paste that password in `/etc/msmtprc`.

#### Install Mail Tools

```
sudo apt install msmtp msmtp-mta mailutils -y
```

---

**Configure SMTP**

Edit:

```
sudo nano /etc/msmtprc
```

Example Gmail configuration:

```
defaults
auth on
tls on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile /var/log/msmtp.log

account gmail
host smtp.gmail.com
port 587
from myemail@gmail.com
user myemail@gmail.com
password YOUR_APP_PASSWORD

account default : gmail
```

Secure file:

```
sudo chmod 600 /etc/msmtprc
```

Test email:

```
echo "Snort email test" | msmtp myemail@gmail.com
```

---

### Create Email Alert Script

```
sudo nano /usr/local/bin/snort-mail.sh
```

Add:

```bash
#!/bin/bash

LOGFILE="/var/log/snort/alert"

tail -Fn0 $LOGFILE | while read line
do

if echo "$line" | grep -q "\[**\]"; then

TIME=$(echo $line | awk '{print $1}')
ATTACK=$(echo $line | sed -n 's/.*] \(.*\) \[\*\*\].*/\1/p')

read proto src arrow dst <<< $(tail -n1 $LOGFILE | grep '{' | awk '{print $3,$4,$5,$6}')

SRCIP=$(echo $src | cut -d':' -f1)
DSTIP=$(echo $dst | cut -d':' -f1)
DSTPORT=$(echo $dst | cut -d':' -f2)

MESSAGE="
⚠️ SNORT IDS ALERT

Time: $TIME
Attack Type: $ATTACK
Protocol: $proto

Attacker IP: $SRCIP
Target IP: $DSTIP
Target Port: $DSTPORT

This activity was detected by the Snort IDS system.

Possible actions:
- Investigate the attacker machine
- Block the IP using firewall
- Check system logs for compromise
"

echo "$MESSAGE" | mail -s "⚠️ Snort Intrusion Alert" myemail@gmail.com

fi

done

```

Make executable:

```
sudo chmod +x /usr/local/bin/snort-mail.sh
```

Run:

```
sudo /usr/local/bin/snort-mail.sh
```

Now every new Snort alert sends an email.

---

### Full Attack Flow Summary

1. Kali runs ping or Nmap scan

2. Snort detects suspicious traffic

3. Alert is written to log file

4. Auto-block script reads alert

5. Attacker IP is blocked using UFW

6. Email notification is sent


---

