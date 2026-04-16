# Free-VPN-OpenVPN-Server-Setup-on-AWS-EC2-Ubuntu


# 🚀 OpenVPN Setup on AWS EC2 (Ubuntu) – Complete DevOps Guide

> Production-ready OpenVPN setup using **Username + Password (PAM Authentication)**  
> Designed for secure remote access for developers (6–15 users)

---

## 🏗️ Architecture Diagram

```mermaid
flowchart LR
    A[Developer Laptop / Mobile] -->|VPN Connection (UDP 1194)| B(OpenVPN Server - AWS EC2)
    B --> C[Internet Access via NAT]
    B --> D[Internal Servers / Hospital Network]

    subgraph AWS Cloud
        B
    end

    subgraph Clients
        A
    end

👉 Explanation:

Users connect via VPN to EC2
EC2 routes traffic to internet/internal systems
NAT ensures proper outbound connectivity
📌 Overview
Server: Ubuntu 22.04 (AWS EC2)
VPN Port: UDP 1194
VPN Network: 10.8.0.0/24
Authentication: Linux users (PAM)
Max Users: ~15 developers
⚙️ Step 0: AWS Pre-Configuration
🔐 Security Group

Allow:

UDP 1194 (VPN)
TCP 22 (SSH – only your IP)

👉 Reason: Allow VPN traffic securely while restricting admin access.

🔄 Disable Source/Destination Check

👉 Reason: Required for routing VPN traffic through EC2.

🧰 Step 1: Install OpenVPN
sudo apt update && sudo apt upgrade -y
sudo apt install openvpn easy-rsa -y

👉 Reason: Install VPN service + certificate tools.

🔐 Step 2: Setup PKI
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
nano vars

Add:

set_var EASYRSA_REQ_COUNTRY    "IN"
set_var EASYRSA_REQ_PROVINCE   "Uttar Pradesh"
set_var EASYRSA_REQ_CITY       "Meerut"
set_var EASYRSA_REQ_ORG        "Demo Hospital"

👉 Reason: Defines certificate identity for secure TLS communication.

🔑 Step 3: Generate Certificates
./easyrsa init-pki
./easyrsa build-ca
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-dh
openvpn --genkey secret ~/openvpn-ca/pki/ta.key

👉 Reason:

CA → trust root
Server cert → server identity
DH → encryption security
ta.key → extra protection
📂 Step 4: Move Certificates
sudo cp ~/openvpn-ca/pki/* /etc/openvpn/ -r

👉 Reason: OpenVPN reads configs from /etc/openvpn/.

⚙️ Step 5: Configure OpenVPN
sudo nano /etc/openvpn/server.conf

Paste:

port 1194
proto udp
dev tun

ca ca.crt
cert server.crt
key server.key
dh dh.pem

tls-auth ta.key 0

server 10.8.0.0 255.255.255.0

push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 1.1.1.1"

cipher AES-256-CBC
data-ciphers AES-256-GCM:AES-128-GCM:AES-256-CBC

auth SHA256

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

👉 Reason: Defines VPN behavior, security, authentication.

🔄 Step 6: Enable IP Forwarding
sudo nano /etc/sysctl.conf

Uncomment:

net.ipv4.ip_forward=1

Apply:

sudo sysctl -p

👉 Reason: Allows routing between VPN clients and network.

🌐 Step 7: Configure NAT
ip route | grep default

Example: ens5

sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/8 -o ens5 -j MASQUERADE
sudo apt install iptables-persistent -y
sudo netfilter-persistent save

👉 Reason: Enables internet access for VPN users.

👤 Step 8: Create Users
sudo useradd -M -s /usr/sbin/nologin dev1
sudo passwd dev1

👉 Reason: Each developer gets login credentials.

▶️ Step 9: Start OpenVPN
sudo systemctl start openvpn@server
sudo systemctl enable openvpn@server
sudo systemctl status openvpn@server

👉 Reason: Start service + auto-start on reboot.

💻 Step 10: Client Config
cat > client.ovpn << EOF
client
dev tun
proto udp
remote YOUR_PUBLIC_IP 1194
auth-user-pass
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-CBC
verb 3
EOF

👉 Reason: Defines client connection settings.

🔐 Step 11: Embed Certificates
echo "<ca>" >> client.ovpn
sudo cat /etc/openvpn/ca.crt >> client.ovpn
echo "</ca>" >> client.ovpn

echo "<tls-auth>" >> client.ovpn
sudo cat /etc/openvpn/ta.key >> client.ovpn
echo "</tls-auth>" >> client.ovpn

👉 Reason: Single file setup for easy sharing.

📥 Step 12: Download Config
scp -i key.pem ubuntu@YOUR_PUBLIC_IP:~/client.ovpn .

👉 Reason: Transfer securely to local system.

📱 Step 13: Connect VPN
Platform	App
Windows	OpenVPN GUI
macOS	Tunnelblick
Mobile	OpenVPN Connect

👉 Import .ovpn → Login → Connected ✅

🛠️ Troubleshooting

TLS Error

Cause: Cipher mismatch
Fix: Add data-ciphers

Server Not Starting

Cause: Missing cert
Fix:
./easyrsa sign-req server server
🔧 Maintenance
sudo systemctl status openvpn@server
sudo tail -f /var/log/openvpn/openvpn.log
sudo systemctl restart openvpn@server
⚠️ Best Practices
Use Elastic IP
Rotate passwords
Monitor EC2 performance
Backup /etc/openvpn
Enable log rotation
🎯 Final Result

✔ Secure VPN Access
✔ Remote Developer Connectivity
✔ PAM Authentication
✔ Production Ready

👨‍💻 Author

Anuj Kumar
DevOps Engineer
