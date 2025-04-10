# DNS Server Setup in Master-Slave Mode using BIND9

## 📘 Overview
This guide covers the complete setup and understanding of a Master-Slave DNS Server configuration using BIND9 on Ubuntu (or Debian-based systems). It also includes advanced implementations such as adding new zones, configuring reverse DNS, and subdomains.

---

## 🧠 DNS Concepts

- **DNS (Domain Name System)**: Translates human-readable domain names (e.g., google.com) into IP addresses.
- **BIND9**: Berkeley Internet Name Domain is the most commonly used DNS server software.
- **Master Server**: The authoritative DNS server where zone files are created and edited.
- **Slave Server**: A read-only copy of the master DNS. It syncs using zone transfers.
- **Zone**: A portion of the DNS namespace managed by a DNS server.
- **Zone Transfer**: Mechanism by which slave servers obtain data from the master server.

---

## 🖥️ Setup Requirements
- Two machines (can be VMs): One as **Master**, one as **Slave**.
- OS: Ubuntu/Debian
- Static IPs assigned:
  - Master: `192.168.56.10`
  - Slave: `192.168.56.11`
- Domain: `nwlab.cse.nitc.ac.in`

---

## 🔧 Step-by-Step Configuration

### 🔹 1. Install BIND9
```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc dnsutils -y
```

---
### 🔹 1.1 Create Zones Directory:
```bash
sudo mkdir /etc/bind/zones

```
### 🔹 1.2 Define zone
```bash
sudo nano /etc/bind/named.conf.local
```
#### ➤ Named Config: Edit named.conf.local add this

```bash
zone "nwlab.cse.nitc.ac.in" {
    type master;
    file "/etc/bind/zones/db.nwlab.cse.nitc.ac.in";
    allow-transfer { 192.168.56.11; };
};
```
### 🔹 1.3 Create zone files
```bash
sudo cp /etc/bind/db.local /etc/bind/zones/db.nwlab.cse.nitc.ac.in
sudo nano /etc/bind/zones/db.nwlab.cse.nitc.ac.in
```

### 🔹 2. Configure the Master Server
#### ➤ Edit Zone File:

Paste:
```bash
$TTL    604800
@       IN      SOA     ns1.nwlab.cse.nitc.ac.in. root.nwlab.cse.nitc.ac.in. (
                             2025040701 ; Serial (MUST update each time you add a entry to manage versions/updates in dns entry)
                             604800     ; Refresh
                             86400      ; Retry
                             2419200    ; Expire
                             604800 )   ; Negative Cache TTL

; Name servers
@       IN      NS      ns1.nwlab.cse.nitc.ac.in.
@       IN      NS      ns2.nwlab.cse.nitc.ac.in.

; A records
ns1     IN      A       192.168.56.10
ns2     IN      A       192.168.56.11
host1   IN      A       192.168.56.20

; CNAME or additional entries if needed
```

#### ➤ Allow Listening: can skip by turning off the automatic dns server while setting ip.
Edit `/etc/bind/named.conf.options`:
```bash
options {
    directory "/var/cache/bind";
    listen-on { any; };
    allow-query { any; };
};
```

#### ➤ Restart BIND:
```bash
sudo named-checkzone nwlab.cse.nitc.ac.in /etc/bind/zones/db.nwlab.cse.nitc.ac.in
sudo systemctl restart bind9
```
#### ➤ Or Reload Server without Restart
```bash
sudo rndc reload
```
---

#### 🔹 3. Configure the Slave Server

#### ➤ Edit Named Config:
```bash
sudo nano /etc/bind/named.conf.local
```
Paste:
```bash
zone "nwlab.cse.nitc.ac.in" {
    type slave;
    masters { 192.168.56.10; };
    file "/var/cache/bind/db.nwlab.cse.nitc.ac.in";
};
```

#### ➤ Restart BIND:
```bash
sudo systemctl restart bind9
```
#### ➤ Or Reload Server without Restart
```bash
sudo rndc reload
```
---

## 🔍 Testing Your Setup
## Replace localhost by dns server ip and host domain to be resolved 
### ✔️ From Any Host:
```bash
nslookup www.nwlab.cse.nitc.ac.in 192.168.56.10
nslookup www.nwlab.cse.nitc.ac.in 192.168.56.11
```

### ✔️ Locally:
```bash
dig @localhost www.nwlab.cse.nitc.ac.in
```

### ✔️ Check if Zone Transferred:
```bash
ls /var/cache/bind/ | grep nwlab
```

----------------

## ⚙️ Advanced Implementations

### 🔸 1. Add a New DNS Zone (Virtual Domain)

#### ➤ Master:
```bash
zone "test.nitc.ac.in" {
    type master;
    file "/etc/bind/zones/db.test.nitc.ac.in";
    allow-transfer { 192.168.56.11; };
};
```
Create zone file:
```bash
sudo nano /etc/bind/zones/db.test.nitc.ac.in
```
```bash
$TTL 604800
@ IN SOA ns1.test.nitc.ac.in. admin.test.nitc.ac.in. (
2025040701 604800 86400 2419200 604800 )
@ IN NS ns1.test.nitc.ac.in.
ns1 IN A 192.168.56.10
www IN A 192.168.56.200
```

#### ➤ Slave:
```bash
zone "test.nitc.ac.in" {
    type slave;
    masters { 192.168.56.10; };
    file "/var/cache/bind/db.test.nitc.ac.in";
};
```

#### ✔️ Test:
```bash
nslookup www.test.nitc.ac.in 192.168.56.10
```

---

### 🔸 2. Configure Reverse DNS (PTR Records)

#### ➤ Master:
```bash
zone "56.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192";
    allow-transfer { 192.168.56.11; };
};
```

#### ➤ Create PTR File:
```bash
sudo nano /etc/bind/zones/db.192
```
```bash
$TTL 604800
@ IN SOA ns1.nwlab.cse.nitc.ac.in. admin.nwlab.cse.nitc.ac.in. (
2025040701 604800 86400 2419200 604800 )
@ IN NS ns1.nwlab.cse.nitc.ac.in.
10 IN PTR ns1.nwlab.cse.nitc.ac.in.
100 IN PTR www.nwlab.cse.nitc.ac.in.
```

#### ✔️ Test Reverse Lookup:
```bash
nslookup 192.168.56.100
```

---

### 🔸 3. Add a Subdomain

#### ➤ Update Zone File on Master:
```bash
sudo nano /etc/bind/zones/db.nwlab.cse.nitc.ac.in
```
Add:
```bash
host1 IN A 192.168.56.150
```

#### ✔️ Test:
```bash
nslookup host1.nwlab.cse.nitc.ac.in 192.168.56.10
```

---

## 🛠️ Troubleshooting Tips

- Always use:
```bash
sudo named-checkzone yourdomain /path/to/zonefile
```
- Restart after changes:
```bash
sudo systemctl restart bind9
```
- Check logs:
```bash
sudo tail -n 50 /var/log/syslog | grep named
```
- Check if slave synced:
```bash
ls /var/cache/bind/
```

---
## 🧠 Why Is There a Specific Directory/File Naming Convention in BIND?

Because BIND (named) is a configuration-heavy service, it:
###	•	expects files in certain locations (based on OS defaults or /etc/named.conf).
###	•	uses standard naming to clearly identify zone files and role (master/slave).
###	•	follows security and permission best practices for DNS data storage.
```bash
/etc/
├── named.conf
├── named.rfc1912.zones

/var/named/
├── master/
│   ├── forward.zone       # Master forward zone
│   └── reverse.zone       # Master reverse zone
├── slaves/
│   └── forward.zone       # Copied from master
```
## 📚 Resources
- BIND9 Manual: `man named`
- [ISC BIND Documentation](https://bind9.readthedocs.io/)

---

## 🧾 Author
Akash Kumar  
Assignment: DNS Server Master-Slave Configuration  
Year: 2025

