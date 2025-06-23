---
title: "Iptables/Firewalld + ipset Basics and Blocking Malicious Ips from Abuseip Dataset"
date: 2025-06-23 12:26:21 +0300
categories: [Scripting, bash, Blue-Team]
tags: [virtualization, scripting, bash]
image: /assets/img/firewalld _ iptabes.png
alt: "Iptables/Firewalld + ipset Basics and Blocking Malicious Ips from Abuseip Dataset"
---

# FIREWALLD

### Install firewalld


### Basic commands
```bash
# Start / Enable / Check status
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo systemctl status firewalld

# Reload config (after changes)
sudo firewall-cmd --reload
```
### Zones
```bash 
# List all zones
firewall-cmd --get-zones

# List zones for a specific interfact
firewall-cmd --get-zone-of-interface=eth0

# Show default zone
firewall-cmd --get-default-zone

# List all zone configs
firewall-cmd --list-all-zones

# List active zones
firewall-cmd --get-active-zones

# Set default zone
sudo firewall-cmd --set-default-zone=home

# Assign interface to a zone
sudo firewall-cmd --zone=home --change-interface=eth0 --permanent
```

### Services 
```bash
# List all available services
firewall-cmd --get-services

# List services in a specific zone
firewall-cmd --list-services --zone=public

# Allow a service
sudo firewall-cmd --zone=public --add-service=http
sudo firewall-cmd --zone=public --add-service=http --permanent

# Remove a service
sudo firewall-cmd --zone=public --remove-service=http --permanent
```

### Ports
```bash
# Open a TCP or UDP port
sudo firewall-cmd --zone=public --add-port=8080/tcp
sudo firewall-cmd --zone=public --add-port=53/udp --permanent

# Close a port
sudo firewall-cmd --zone=public --remove-port=8080/tcp --permanent

# List open ports in current zone
firewall-cmd --list-ports
```
### Interfaces
```bash
# List interfaces and their zones
firewall-cmd --get-active-zones

# Change zone for an interface
sudo firewall-cmd --zone=internal --change-interface=eth0 --permanent
```
### Immediate vs Permanent

`--permanent` → Saves rule across reboots

`No --permanent` → Temporary until next reload or reboot

### Masquerading / NAT
```bash
# Enable masquerading (useful for routers)
sudo firewall-cmd --zone=external --add-masquerade --permanent
```
### Useful Combinations
```bash
# Allow a service and limit to a single IP
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="1.2.3.4" service name="ssh" accept'

# Drop all traffic from a subnet
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" drop'
```
### List Services Running locally
```
ss -tuln
netstat -tuln
```
### Blocking IPs
```bash
# Block a single IP (rich rule)
sudo firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=1.2.3.4 reject'

# Drop instead of reject
sudo firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=1.2.3.4 drop'

# Remove rich rule
sudo firewall-cmd --permanent --remove-rich-rule='rule family=ipv4 source address=1.2.3.4 reject'

# View all rich rules
firewall-cmd --list-rich-rules

```
### Automation
Script to scan open ports and auto-whitelist them in firewalld

```bash
for port in $(ss -tuln | awk 'NR>1 {print $5}' | cut -d: -f2 | sort -n | uniq); do
  firewall-cmd --permanent --add-port=${port}/tcp
done
firewall-cmd --reload
```

### To list specific services that are already running

```
sudo ss -tulnp | awk -F '[",]' 'NR>1 && /users/ { print $2 }' | sort -u
```
Now automating adding them to the whitelist:
```bash
for i in $(sudo ss -tulnp | awk -F '[",]' 'NR>1 && /users/ { print $2 }' | sort -u); do
 firewall-cmd --permanent --zone=public --add-service=$i;
 done
```

### For Custom services running on dynamic IPs like mosh (for ssh) 
Create firewalld services in xml formats in the path /etc/firewalld/services/service.xml
Using a mosh example, it dynamically assigns ip addresses between 60000 and 61000

```
nano /etc/firewalld/services/mosh.xml
```
```xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>Mosh service</short>
  <description>mosh service</description>
  <port protocol="udp" port="60000-61000"/>
</service>
```
```
firewall-cmd --permanent --add-service=mosh
firewall-cmd --reload
```
Now your mosh connection should work well.

### Blocking Large datasets 
Not advicable. Will slow down the server since its not optimized for that. Consider using iptables and ipset.

 |<br>
 |<br>
 |<br>
 |<br>
 |<br>
 |<br>
\\/
 
# IPTABLES and IPSET 
### Installation
```bash
sudo apt install ipset 
sudo apt install iptables
```
Allow a service e.g ssh
```sh
iptables -A INPUT -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
```
Allow HTTP and HTTPS
```sh
iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT
```
Block an ip
```sh
iptables -A INPUT -s 111.1.111.1 -j DROP
```
Block All Incoming Traffic (after allowing specific ports)
```sh
iptables -P INPUT DROP
iptables -P FORWARD DROP
```
Allow Established & Related Connections
```sh
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```
Allow localhost traffic
```sh
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT
```
Persistant saving of rules
```sh
apt install iptables-persistent
netfilter-persistent save
```
Manually save rules
```sh
iptables-save > /etc/iptables/rules.v4
```
Restore rules
```sh
iptables-restore < /etc/iptables/rules.v4
```
Limit SSH to 3 connections per minute
```sh
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --set
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --update --seconds 60 --hitcount 3 -j DROP
```
Block Specific Countries with ipset + geoip
```sh
iptables -A INPUT -m geoip --src-cc CN,RU -j DROP
```
Enable NAT (masquerading) for internal network
```sh
iptables -t nat -A POSTROUTING -o eth0 -s 192.168.1.0/24 -j MASQUERADE

# Also enable ip forwarding 
echo 1 > /proc/sys/net/ipv4/ip_forward
```
Redirect(proxy) HTTP traffic to local Squid on port 3128
```sh
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 3128
```
Block IP Range or CIDR
```sh
iptables -A INPUT -s 203.0.113.0/24 -j DROP
```
Use iptables-restore in safe test mode via at (just so kasinuke ile serious)
```sh
echo "iptables-restore < ~/rules.v4" | at now + 2 minutes
```
<br>

# Case Study to block 100+ ips from abuseip database
Doing this prcess using a bash script to import an IP at a time is consuming and may end up caking up to an hour of our time.

Best bet would be doing a restore using ipset. Here is how:
But before, Lets look at the currently existing iptable rules,
```bash
iptables -L -n -v
```
Set Default Policies to ACCEPT (safe baseline)
```bash
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P OUTPUT ACCEPT
```
Save the current state (optional but smart)

Before making changes:

```bash
iptables-save > ~/iptables-backup.rules
```
You can restore this if something goes wrong:

```bash
iptables-restore < ~/iptables-backup.rules
```
Allow SSH (critical for remote access):
```bash
iptables -A INPUT -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
```
Allow Localhost traffic:
```bash
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
```
Allow Established connections:
```bash
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED -j ACCEPT
```
/------------------------------------------------------------------------

Important: DO NOT set default DROP until SSH (or other admin ports) is allowed, or you'll lock yourself out.

/------------------------------------------------------------------------

## Ipset magic
While `iptables -A INPUT -s 111.1.111.1 -j DROP` works, its not feasible for large datasets. Also ipset has a default cap of just over 65k ips. We need to increase this to handle our many ips.

```bash
# create a large capacity ipset to handle max 200k ips
ipset create blacklist hash:ip maxelem 200000 hashsize 65536
```
Create a file blacklist-malips.txt with one IP per line:
```bash
curl -s  https://raw.githubusercontent.com/borestad/blocklist-abuseipdb/refs/heads/main/abuseipdb-s100-1d.ipv4 |sed /^#/d | awk '{print $1}' > abuseips.txt
```
Script to create an ipset and export the ips from abuseip db to a restore file. If doing this, ignore the step above. 
```bash
#create a ipset called blacklist-malip with a max of 200k ips to it and genetate an export file
(   echo "create blacklist-malip hash:ip family inet hashsize 65536 maxelem 200000";   grep -Eo '([0-9]{1,3}\.){3}[0-9]{1,3}' abuseips.txt | awk '{ print "add blacklist-malip -exist", $1 }'; ) > ipset-batch.conf
```
Perform `ipset restore`
```bash
ipset restore < ipset-batch.conf
```
Check if the ips have been ingested.
```bash
ipset list blacklist-malip | wc -l
```
Use iptables to block traffic from the blacklist
```bash
iptables -I INPUT -m set --match-set blacklist-malip src -j DROP
```
Test and Persist the Rules
```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
```
### Automating the ipset process
```sh
#!/bin/bash
# get ips from abuseips
curl -s  https://raw.githubusercontent.com/borestad/blocklist-abuseipdb/refs/heads/main/abuseipdb-s100-1d.ipv4 |sed /^#/d | awk '{print $1}' > abuseips.txt

# export the ips to a restore file
( grep -Eo '([0-9]{1,3}\.){3}[0-9]{1,3}' abuseips.txt | awk '{ print "add blacklist-malip -exist", $1 }'; ) > ipset-batch.conf

#update the ipset
ipset restore < ipset-batch.conf
```
Save for a file `block_the_abuser.sh`
chmod +x `block_the_abuser.sh`

### Cron the process to update every 2 days
Edit crontab
```bash
crontab -e
```
Add entry
```bash
0 2 */2 * * /path/to/your/block_the_abuser.sh
```
Save and exit.

With this simple process, we have effectively used Iptables + ipset to block malicious ips from accessing out servers. This is not all the protection in the world, but a good ground for blocking known attackers scanning the internet for vulnerabilities.
