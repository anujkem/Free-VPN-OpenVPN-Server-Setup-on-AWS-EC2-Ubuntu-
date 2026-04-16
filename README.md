# OpenVPN Server Setup on AWS EC2

> Complete step-by-step guide to deploy a self-hosted OpenVPN server on AWS EC2 with username/password authentication for remote developer access.

---

## Table of Contents

- [Overview](#overview)
- [Pre-Requisites](#pre-requisites)
- [Installation Steps](#installation-steps)
- [Client Configuration](#client-configuration)
- [Troubleshooting](#troubleshooting)
- [Ongoing Maintenance](#ongoing-maintenance)
- [Future Issues & Prevention](#future-issues--prevention)
- [Quick Reference](#quick-reference)

---

## Overview

This guide sets up a production-ready OpenVPN server on an AWS EC2 Ubuntu instance. Developers connect using a shared `.ovpn` config file with individual username/password credentials authenticated via Linux PAM.

| Field | Detail |
|-------|--------|
| **Server OS** | Ubuntu 20.04 / 22.04 LTS |
| **Hosting** | AWS EC2 |
| **VPN Software** | OpenVPN 2.x |
| **Auth Method** | Username + Password (PAM) |
| **VPN Subnet** | `10.8.0.0/24` |
| **Port** | UDP `1194` |
| **Encryption** | AES-256-GCM (Data), TLS 1.3 (Control) |

---

## Pre-Requisites

### 1. AWS EC2 Security Group Rules

Add the following **Inbound Rules** to your EC2 Security Group:

| Type | Protocol | Port | Source | Purpose |
|------|----------|------|--------|---------|
| Custom UDP | UDP | 1194 | 0.0.0.0/0 | OpenVPN client connections |
| SSH | TCP | 22 | Your IP only | Server management |

> ⚠️ **Never expose SSH (port 22) to 0.0.0.0/0 in production.**

---

### 2. Disable EC2 Source/Destination Check

This step is **mandatory** — without it, VPN traffic cannot be routed through the instance.

- AWS Console → EC2 → Instances → Select your instance
- **Actions → Networking → Change source/destination check → Disable** ✅

---

### 3. SSH into Your EC2 Instance

```bash
ssh -i your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

---

## Installation Steps

### Step 1 — System Update & Install Packages

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install openvpn easy-rsa -y
```

---

### Step 2 — EasyRSA PKI Setup

```bash
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
nano vars
```

Update the following values in `vars`:

```bash
set_var EASYRSA_REQ_COUNTRY    "IN"           # Your country code
set_var EASYRSA_REQ_PROVINCE   "Your Province"
set_var EASYRSA_REQ_CITY       "Your City"
set_var EASYRSA_REQ_ORG        "Your Organization"
set_var EASYRSA_REQ_EMAIL      "admin@yourdomain.com"
set_var EASYRSA_REQ_OU         "IT Department"
set_var EASYRSA_KEY_SIZE       2048
set_var EASYRSA_CA_EXPIRE      3650
set_var EASYRSA_CERT_EXPIRE    825
```

---

### Step 3 — Generate Certificates

```bash
# Initialize PKI
./easyrsa init-pki

# Build CA — set a strong passphrase and store it securely
./easyrsa build-ca

# Generate server cert request (enter "server" as Common Name)
./easyrsa gen-req server nopass

# Sign the server certificate — type 'yes', then enter CA passphrase
./easyrsa sign-req server server

# Generate Diffie-Hellman parameters (takes 1-2 minutes)
./easyrsa gen-dh

# Generate TLS Auth key
openvpn --genkey secret ~/openvpn-ca/pki/ta.key
```

> ⚠️ **Store your CA passphrase securely (password manager). You will need it to sign certificates in the future.**

---

### Step 4 — Copy Certificates to OpenVPN Directory

```bash
sudo cp ~/openvpn-ca/pki/ca.crt /etc/openvpn/
sudo cp ~/openvpn-ca/pki/issued/server.crt /etc/openvpn/
sudo cp ~/openvpn-ca/pki/private/server.key /etc/openvpn/
sudo cp ~/openvpn-ca/pki/dh.pem /etc/openvpn/
sudo cp ~/openvpn-ca/pki/ta.key /etc/openvpn/
```

Verify all files are present:

```bash
sudo ls -la /etc/openvpn/
```

Expected: `ca.crt`, `server.crt`, `server.key`, `dh.pem`, `ta.key`

---

### Step 5 — Create Server Configuration

```bash
sudo nano /etc/openvpn/server.conf
```

Paste the following:

```conf
port 1194
proto udp
dev tun

ca ca.crt
cert server.crt
key server.key
dh dh.pem
tls-auth ta.key 0

server 10.8.0.0 255.255.255.0
ifconfig-pool-persist /var/log/openvpn/ipp.txt

push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 1.1.1.1"

keepalive 10 120
cipher AES-256-CBC
data-ciphers AES-256-GCM:AES-128-GCM:AES-256-CBC
data-ciphers-fallback AES-256-CBC
auth SHA256

# Username + Password auth via PAM
plugin /usr/lib/openvpn/openvpn-plugin-auth-pam.so login
verify-client-cert none
username-as-common-name

user nobody
group nogroup
persist-key
persist-tun

status /var/log/openvpn/openvpn-status.log
log-append /var/log/openvpn/openvpn.log
verb 3
max-clients 15
```

```bash
# Create log directory
sudo mkdir -p /var/log/openvpn
```

---

### Step 6 — Enable IP Forwarding

```bash
sudo nano /etc/sysctl.conf
```

Uncomment this line (remove the `#`):

```
net.ipv4.ip_forward=1
```

Apply immediately:

```bash
sudo sysctl -p
```

---

### Step 7 — NAT / iptables Rules

First, find your network interface name:

```bash
ip route | grep default
# Example output: default via 172.31.x.x dev ens5 ...
```

> On AWS EC2, the interface is typically `ens5`. On other platforms it may be `eth0` or `ens3`.

```bash
# Apply NAT masquerade — replace ens5 with your interface
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/8 -o ens5 -j MASQUERADE

# Make persistent across reboots
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

Verify the rule is active:

```bash
sudo iptables -t nat -L POSTROUTING -v --line-numbers
```

> 💡 This guide uses `iptables` directly because UFW was inactive. If UFW is active on your server, add the NAT rule to `/etc/ufw/before.rules` instead and set `DEFAULT_FORWARD_POLICY="ACCEPT"` in `/etc/default/ufw`.

---

### Step 8 — Create Developer Users

Each developer gets a Linux system user with no shell and no home directory:

```bash
sudo useradd -M -s /usr/sbin/nologin dev1
sudo passwd dev1

sudo useradd -M -s /usr/sbin/nologin dev2
sudo passwd dev2

# Repeat for all developers
```

Verify:

```bash
cat /etc/passwd | grep dev
```

| Flag | Meaning |
|------|---------|
| `-M` | No home directory |
| `-s /usr/sbin/nologin` | No SSH shell — VPN access only |

---

### Step 9 — Start OpenVPN Service

```bash
sudo systemctl start openvpn@server
sudo systemctl enable openvpn@server
sudo systemctl status openvpn@server
```

✅ Expected output: `active (running)` — `Initialization Sequence Completed`

---

## Client Configuration

### Generate client.ovpn File

```bash
cd ~

# Create base config — replace YOUR_SERVER_PUBLIC_IP
cat > client.ovpn << 'EOF'
client
dev tun
proto udp
remote YOUR_SERVER_PUBLIC_IP 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-CBC
auth SHA256
key-direction 1
auth-user-pass
verb 3
EOF

# Embed CA certificate
echo "<ca>" >> client.ovpn
sudo cat /etc/openvpn/ca.crt >> client.ovpn
echo "</ca>" >> client.ovpn

# Embed TLS auth key
echo "<tls-auth>" >> client.ovpn
sudo cat /etc/openvpn/ta.key >> client.ovpn
echo "</tls-auth>" >> client.ovpn
```

### Download to Your Local Machine

```bash
# Run this on your LOCAL machine
scp -i your-key.pem ubuntu@YOUR_SERVER_PUBLIC_IP:~/client.ovpn ~/Downloads/client.ovpn
```

> 💡 All developers share the **same `client.ovpn` file**. Each person authenticates with their own username/password.

---

### Client Software by Platform

| Platform | Software | How to Connect |
|----------|----------|----------------|
| **Windows** | [OpenVPN GUI](https://openvpn.net/community-downloads/) | Right-click tray icon → Import file → Connect → Enter credentials |
| **macOS** | [Tunnelblick](https://tunnelblick.net/downloads.html) | Double-click `.ovpn` → Import → Connect → Enter credentials |
| **Android / iOS** | OpenVPN Connect (App Store / Play Store) | Import `.ovpn` → Connect → Enter credentials |
| **Linux** | `sudo apt install openvpn` | `sudo openvpn --config client.ovpn` |

---

## Troubleshooting

### View Live Server Logs

```bash
sudo tail -f /var/log/openvpn/openvpn.log
```

### Check Connected Clients

```bash
sudo cat /var/log/openvpn/openvpn-status.log
```

---

### Common Errors

#### `server.crt` Missing — Service Fails to Start

```
openvpn@server.service failed — exit code 1
```

**Cause:** `easyrsa sign-req` was not run, so `server.crt` was never generated.

**Fix:**
```bash
cd ~/openvpn-ca
./easyrsa sign-req server server
sudo cp ~/openvpn-ca/pki/issued/server.crt /etc/openvpn/
sudo systemctl start openvpn@server
```

---

#### TLS Handshake Timeout (Client Side)

```
TLS Error: TLS key negotiation failed to occur within 60 seconds
TLS Error: TLS handshake failed
```

**Cause 1:** UDP port 1194 not open in EC2 Security Group → Add inbound rule.

**Cause 2:** Cipher mismatch — newer clients use `AES-256-GCM`, older server config only had `AES-256-CBC`.

**Fix:** Add to `/etc/openvpn/server.conf`:
```conf
data-ciphers AES-256-GCM:AES-128-GCM:AES-256-CBC
data-ciphers-fallback AES-256-CBC
```
```bash
sudo systemctl restart openvpn@server
```

---

#### No Traffic After Connect

**Cause:** IP forwarding is off or NAT rule is missing.

**Fix:**
```bash
# Check forwarding
cat /proc/sys/net/ipv4/ip_forward   # Should be 1

# Check NAT rule
sudo iptables -t nat -L POSTROUTING -v

# Re-apply if missing
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/8 -o ens5 -j MASQUERADE
sudo netfilter-persistent save
```

---

## Ongoing Maintenance

### Useful Commands

```bash
# Service management
sudo systemctl status openvpn@server
sudo systemctl restart openvpn@server
sudo systemctl stop openvpn@server

# User management
sudo useradd -M -s /usr/sbin/nologin newdev   # Add user
sudo passwd newdev                             # Set password
sudo passwd -l username                        # Lock user (immediate)
sudo userdel username                          # Delete user

# Logs
sudo tail -f /var/log/openvpn/openvpn.log
sudo cat /var/log/openvpn/openvpn-status.log

# iptables
sudo iptables -t nat -L POSTROUTING -v
sudo netfilter-persistent save
```

### Log Rotation

Prevent log files from growing too large:

```bash
sudo nano /etc/logrotate.d/openvpn
```

```
/var/log/openvpn/*.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
}
```

---

## Future Issues & Prevention

### 1. Server Certificate Expiry (~825 days)

- **Problem:** All clients suddenly get TLS errors
- **Prevention:** Set a calendar reminder 30 days before expiry

```bash
cd ~/openvpn-ca
./easyrsa renew server nopass
sudo cp ~/openvpn-ca/pki/issued/server.crt /etc/openvpn/
sudo systemctl restart openvpn@server
```

---

### 2. EC2 Public IP Changes on Restart

- **Problem:** Stopping/starting EC2 assigns a new public IP — `client.ovpn` breaks
- **Prevention:** Assign an **AWS Elastic IP** to your instance (free while instance is running)
- **Fix if it happens:** Update `remote` IP in `client.ovpn` and redistribute

---

### 3. Credentials Compromised

```bash
sudo passwd -l compromised_user   # Lock immediately
sudo userdel compromised_user     # Remove permanently
```

No need to reissue `client.ovpn` to other users.

---

### 4. iptables Rules Lost After Reboot

```bash
# Verify after reboot
sudo iptables -t nat -L POSTROUTING -v

# Re-apply and save if missing
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/8 -o ens5 -j MASQUERADE
sudo netfilter-persistent save
```

---

### 5. Performance Under Load

- **Problem:** Small EC2 instances (t2.micro) may struggle with many concurrent users
- **Fix:** Upgrade instance type — `t3.small` handles ~20 concurrent VPN users comfortably

---

### 6. OpenVPN Client Cipher Deprecation

- **Problem:** Future clients may drop legacy cipher support
- **Prevention:** Keep `data-ciphers` line in `server.conf` — already supports `AES-256-GCM`
- **Keep server updated:**

```bash
sudo apt update && sudo apt upgrade openvpn -y
```

---

## Quick Reference

| Item | Value |
|------|-------|
| **VPN Port** | UDP `1194` |
| **VPN Subnet** | `10.8.0.0/24` |
| **Client IP Range** | `10.8.0.4` – `10.8.0.65` |
| **Server Config** | `/etc/openvpn/server.conf` |
| **Certificates** | `/etc/openvpn/` |
| **PKI Directory** | `~/openvpn-ca/pki/` |
| **Log File** | `/var/log/openvpn/openvpn.log` |
| **Status File** | `/var/log/openvpn/openvpn-status.log` |
| **Client Config** | `~/client.ovpn` |
| **Windows Client** | [OpenVPN GUI](https://openvpn.net/community-downloads/) |
| **macOS Client** | [Tunnelblick](https://tunnelblick.net/downloads.html) |
| **Mobile Client** | OpenVPN Connect (App Store / Play Store) |

---

## License

MIT — feel free to use and adapt this guide for your own infrastructure.
