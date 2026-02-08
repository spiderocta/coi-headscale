# Headscale VPN Setup Guide
## Complete Guide for Office Network Access via OCI Free Tier

### Architecture Overview

```
Users (Home) → Headscale (OCI) → Office Subnet Router → Office Network (ESXi, VMs, etc.)
```

---

## Part 1: Prepare OCI Free Tier VM

### 1.1 Create OCI VM Instance

1. Sign up for Oracle Cloud Free Tier: https://www.oracle.com/cloud/free/
2. Navigate to: **Compute → Instances → Create Instance**
3. Configuration:
   - **Name:** headscale-server
   - **Image:** Ubuntu 22.04 (Always Free Eligible)
   - **Shape:** VM.Standard.E2.1.Micro (Always Free)
   - **Network:** Create new VCN or use existing
   - **Public IP:** Assign public IPv4 address
   - **SSH Keys:** Upload or generate SSH key pair

4. **IMPORTANT:** Download your SSH private key!

### 1.2 Configure Security Rules

In OCI Console:
1. Go to **Networking → Virtual Cloud Networks**
2. Select your VCN → Security Lists → Default Security List
3. Add Ingress Rules:

| Source CIDR | Protocol | Port Range | Description |
|-------------|----------|------------|-------------|
| 0.0.0.0/0 | TCP | 22 | SSH |
| 0.0.0.0/0 | TCP | 443 | HTTPS (Headscale) |
| 0.0.0.0/0 | TCP | 8080 | HTTP (Headscale Web UI) |
| 0.0.0.0/0 | UDP | 3478 | DERP/STUN |
| 0.0.0.0/0 | TCP | 80 | HTTP (Let's Encrypt) |

### 1.3 Configure Ubuntu Firewall

SSH into your OCI VM:
```bash
ssh ubuntu@<YOUR_OCI_PUBLIC_IP>
```

Then run:
```bash
sudo ufw allow 22/tcp
sudo ufw allow 443/tcp
sudo ufw allow 8080/tcp
sudo ufw allow 3478/udp
sudo ufw allow 80/tcp
sudo ufw enable
```

---

## Part 2: Install Headscale on OCI VM

### 2.1 Automated Installation Script

Save this script as `install-headscale.sh`:

```bash
#!/bin/bash

set -e

echo "================================================"
echo "Headscale Installation Script for OCI Ubuntu"
echo "================================================"

# Update system
echo "[1/8] Updating system packages..."
sudo apt update && sudo apt upgrade -y

# Install required packages
echo "[2/8] Installing dependencies..."
sudo apt install -y wget curl git sqlite3 apache2-utils

# Download latest Headscale
echo "[3/8] Downloading Headscale..."
HEADSCALE_VERSION="0.23.0"
wget "https://github.com/juanfont/headscale/releases/download/v${HEADSCALE_VERSION}/headscale_${HEADSCALE_VERSION}_linux_amd64.deb"

# Install Headscale
echo "[4/8] Installing Headscale..."
sudo dpkg -i "headscale_${HEADSCALE_VERSION}_linux_amd64.deb"

# Create necessary directories
echo "[5/8] Creating directories..."
sudo mkdir -p /etc/headscale
sudo mkdir -p /var/lib/headscale

# Create configuration file
echo "[6/8] Creating configuration..."
read -p "Enter your domain or public IP (e.g., headscale.yourdomain.com or 123.45.67.89): " SERVER_ADDRESS

sudo tee /etc/headscale/config.yaml > /dev/null <<EOF
server_url: http://${SERVER_ADDRESS}:8080
listen_addr: 0.0.0.0:8080
metrics_listen_addr: 127.0.0.1:9090

grpc_listen_addr: 0.0.0.0:50443
grpc_allow_insecure: false

private_key_path: /var/lib/headscale/private.key
noise:
  private_key_path: /var/lib/headscale/noise_private.key

ip_prefixes:
  - fd7a:115c:a1e0::/48
  - 100.64.0.0/10

derp:
  server:
    enabled: true
    region_id: 999
    region_code: "headscale"
    region_name: "Headscale Embedded DERP"
    stun_listen_addr: "0.0.0.0:3478"

  urls:
    - https://controlplane.tailscale.com/derpmap/default

  auto_update_enabled: true
  update_frequency: 24h

disable_check_updates: false
ephemeral_node_inactivity_timeout: 30m

database:
  type: sqlite3
  sqlite:
    path: /var/lib/headscale/db.sqlite

log:
  format: text
  level: info

acl_policy_path: ""

dns_config:
  nameservers:
    - 1.1.1.1
    - 8.8.8.8
  magic_dns: true
  base_domain: headscale.net

unix_socket: /var/run/headscale/headscale.sock
unix_socket_permission: "0770"
EOF

# Set permissions
echo "[7/8] Setting permissions..."
sudo chown -R headscale:headscale /etc/headscale
sudo chown -R headscale:headscale /var/lib/headscale

# Enable and start Headscale
echo "[8/8] Starting Headscale service..."
sudo systemctl enable headscale
sudo systemctl start headscale

# Wait for service to start
sleep 3

# Check status
echo ""
echo "================================================"
echo "Installation Complete!"
echo "================================================"
sudo systemctl status headscale --no-pager

echo ""
echo "Headscale is now running at: http://${SERVER_ADDRESS}:8080"
echo ""
echo "Next steps:"
echo "1. Create a user: sudo headscale users create <username>"
echo "2. Generate pre-auth key: sudo headscale preauthkeys create --user <username> --reusable --expiration 24h"
echo ""
```

Make it executable and run:
```bash
chmod +x install-headscale.sh
./install-headscale.sh
```

### 2.2 Create Users and Pre-Auth Keys

Create a user (e.g., "office"):
```bash
sudo headscale users create office
```

Generate a reusable pre-auth key:
```bash
sudo headscale preauthkeys create --user office --reusable --expiration 720h
```

**SAVE THIS KEY!** You'll need it for all clients.

---

## Part 3: Setup Office Subnet Router

This machine will route traffic to your entire office network.

### 3.1 Install Tailscale on Office Server

For **Ubuntu/Debian**:
```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

For **Windows Server**:
- Download from: https://tailscale.com/download/windows
- Install as a service

For **ESXi** (if you want ESXi itself):
- You'll need to install Tailscale on a VM that can route

### 3.2 Configure Headscale URL

Edit Tailscale to use your Headscale server:

**Linux:**
```bash
sudo tailscale up --login-server=http://<YOUR_OCI_PUBLIC_IP>:8080 --advertise-routes=<OFFICE_SUBNET> --accept-routes --authkey=<YOUR_PREAUTHKEY>
```

Replace:
- `<YOUR_OCI_PUBLIC_IP>`: Your OCI VM's public IP
- `<OFFICE_SUBNET>`: Your office network (e.g., `192.168.1.0/24` or multiple: `192.168.1.0/24,192.168.2.0/24`)
- `<YOUR_PREAUTHKEY>`: The key from Part 2.2

**Windows:**
```powershell
tailscale up --login-server=http://<YOUR_OCI_PUBLIC_IP>:8080 --advertise-routes=<OFFICE_SUBNET> --accept-routes --authkey=<YOUR_PREAUTHKEY>
```

### 3.3 Enable IP Forwarding (Linux)

On the subnet router:
```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 3.4 Approve Routes on Headscale

On your OCI Headscale server:

List nodes:
```bash
sudo headscale nodes list
```

Find your subnet router's ID, then approve routes:
```bash
sudo headscale routes enable -i <NODE_ID> -r <OFFICE_SUBNET>
```

Example:
```bash
sudo headscale routes enable -i 1 -r 192.168.1.0/24
```

Verify:
```bash
sudo headscale routes list
```

---

## Part 4: Setup User Devices

### 4.1 Windows User

1. Download Tailscale: https://tailscale.com/download/windows
2. Install Tailscale
3. Open Tailscale → **Settings** → **Use a different server**
4. Enter: `http://<YOUR_OCI_PUBLIC_IP>:8080`
5. Click **Connect**
6. Or use command line:
```powershell
tailscale up --login-server=http://<YOUR_OCI_PUBLIC_IP>:8080 --accept-routes --authkey=<YOUR_PREAUTHKEY>
```

### 4.2 macOS User

1. Download Tailscale: https://tailscale.com/download/mac
2. Install Tailscale
3. Open Terminal:
```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale up --login-server=http://<YOUR_OCI_PUBLIC_IP>:8080 --accept-routes --authkey=<YOUR_PREAUTHKEY>
```

### 4.3 Linux User

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --login-server=http://<YOUR_OCI_PUBLIC_IP>:8080 --accept-routes --authkey=<YOUR_PREAUTHKEY>
```

### 4.4 Android/iOS

1. Install Tailscale app from store
2. Go to Settings → **Custom control server**
3. Enter: `http://<YOUR_OCI_PUBLIC_IP>:8080`
4. Sign in (you'll need to register via web browser)

**Important:** For mobile, you may need to generate a login URL:
```bash
sudo headscale nodes register --user office --key mkey:<machine_key_from_app>
```

---

## Part 5: Access ESXi and VMs

### 5.1 From User Devices

Once connected, users can access:

- **ESXi Web Interface:** `https://192.168.1.10` (example)
- **VMs via RDP:** `192.168.1.20:3389`
- **VMs via SSH:** `ssh user@192.168.1.21`
- **Any office resource:** Direct IP access

### 5.2 ESXi-Specific Configuration

If you want to manage ESXi VMs directly:

**Option 1: Access via Subnet Router** (Recommended)
- Users access ESXi through the office network routes
- No additional configuration needed

**Option 2: Install Tailscale on a Management VM**
- Create a lightweight VM (Ubuntu/Debian)
- Install Tailscale on it
- Use it as a jump host for ESXi management

---

## Part 6: Management Commands

### 6.1 Headscale Common Commands

**List all nodes:**
```bash
sudo headscale nodes list
```

**List users:**
```bash
sudo headscale users list
```

**Create new user:**
```bash
sudo headscale users create <username>
```

**Generate pre-auth key:**
```bash
sudo headscale preauthkeys create --user <username> --reusable --expiration 720h
```

**List routes:**
```bash
sudo headscale routes list
```

**Enable routes:**
```bash
sudo headscale routes enable -i <node-id> -r <subnet>
```

**Delete node:**
```bash
sudo headscale nodes delete -i <node-id>
```

**View logs:**
```bash
sudo journalctl -u headscale -f
```

### 6.2 Tailscale Client Commands

**Check status:**
```bash
tailscale status
```

**Show IP addresses:**
```bash
tailscale ip
```

**Ping another node:**
```bash
tailscale ping <node-name-or-ip>
```

**Reconnect:**
```bash
sudo tailscale down
sudo tailscale up --login-server=http://<YOUR_OCI_PUBLIC_IP>:8080 --accept-routes
```

---

## Part 7: Troubleshooting

### 7.1 Cannot Connect to Headscale

**Check OCI firewall:**
```bash
sudo ufw status
```

**Check Headscale service:**
```bash
sudo systemctl status headscale
sudo journalctl -u headscale -n 50
```

**Check OCI Security List:**
- Verify ports 8080, 443, 3478 are open in OCI console

### 7.2 Cannot Access Office Network

**Verify routes are enabled:**
```bash
sudo headscale routes list
```

**Check subnet router status:**
```bash
tailscale status
```

**Verify IP forwarding on subnet router:**
```bash
sysctl net.ipv4.ip_forward
sysctl net.ipv6.conf.all.forwarding
```

**Check client accepts routes:**
```bash
tailscale status
# Look for "accept-routes: true"
```

### 7.3 ESXi Access Issues

**Check network connectivity:**
```bash
ping <esxi-ip-from-client>
traceroute <esxi-ip>
```

**Verify ESXi firewall:**
- Ensure ESXi allows connections from Tailscale subnet (100.64.0.0/10)

**Check office router/firewall:**
- May need to allow traffic from subnet router

---

## Part 8: Security Best Practices

### 8.1 Enable HTTPS (Recommended)

**Option 1: Use Caddy (Easiest)**
```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

Create Caddyfile:
```bash
sudo tee /etc/caddy/Caddyfile > /dev/null <<EOF
headscale.yourdomain.com {
    reverse_proxy localhost:8080
}
EOF

sudo systemctl restart caddy
```

**Option 2: Use Let's Encrypt with Certbot**
```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d headscale.yourdomain.com
```

### 8.2 Change Default Ports

Edit `/etc/headscale/config.yaml`:
```yaml
listen_addr: 0.0.0.0:8443  # Change from 8080
```

### 8.3 ACL Rules

Create `/etc/headscale/acl.yaml`:
```yaml
acls:
  - action: accept
    src:
      - "office"
    dst:
      - "192.168.1.0/24:*"
```

Update config.yaml:
```yaml
acl_policy_path: "/etc/headscale/acl.yaml"
```

### 8.4 Regular Updates

```bash
# Update Headscale
wget https://github.com/juanfont/headscale/releases/download/v<VERSION>/headscale_<VERSION>_linux_amd64.deb
sudo dpkg -i headscale_<VERSION>_linux_amd64.deb
sudo systemctl restart headscale

# Update Tailscale clients
sudo tailscale update
```

---

## Part 9: Quick Setup Scripts

### 9.1 Office Subnet Router Setup Script

Save as `setup-subnet-router.sh`:

```bash
#!/bin/bash

set -e

echo "Office Subnet Router Setup"
echo "=========================="

read -p "Enter Headscale server URL (e.g., http://1.2.3.4:8080): " HEADSCALE_URL
read -p "Enter office subnet(s) (e.g., 192.168.1.0/24): " OFFICE_SUBNET
read -p "Enter pre-auth key: " PREAUTH_KEY

echo "Installing Tailscale..."
curl -fsSL https://tailscale.com/install.sh | sh

echo "Enabling IP forwarding..."
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

echo "Connecting to Headscale..."
sudo tailscale up --login-server="${HEADSCALE_URL}" \
  --advertise-routes="${OFFICE_SUBNET}" \
  --accept-routes \
  --authkey="${PREAUTH_KEY}"

echo ""
echo "Setup complete!"
echo "Now approve routes on Headscale server:"
echo "  1. sudo headscale nodes list"
echo "  2. sudo headscale routes enable -i <NODE_ID> -r ${OFFICE_SUBNET}"
```

### 9.2 User Client Setup Script

Save as `setup-client.sh`:

```bash
#!/bin/bash

set -e

echo "Tailscale Client Setup"
echo "====================="

read -p "Enter Headscale server URL (e.g., http://1.2.3.4:8080): " HEADSCALE_URL
read -p "Enter pre-auth key: " PREAUTH_KEY

echo "Installing Tailscale..."
curl -fsSL https://tailscale.com/install.sh | sh

echo "Connecting to Headscale..."
sudo tailscale up --login-server="${HEADSCALE_URL}" \
  --accept-routes \
  --authkey="${PREAUTH_KEY}"

echo ""
echo "Setup complete!"
echo "Check status with: tailscale status"
```

---

## Part 10: Testing the Setup

### 10.1 Verify Connectivity

From user device:

```bash
# Check Tailscale status
tailscale status

# Ping subnet router
tailscale ping <subnet-router-hostname>

# Ping office resource
ping 192.168.1.1

# Test ESXi access
curl -k https://192.168.1.10  # ESXi web interface
```

### 10.2 Speed Test

```bash
# Install iperf3 on subnet router and client
sudo apt install iperf3

# On subnet router
iperf3 -s

# On client
iperf3 -c <subnet-router-tailscale-ip>
```

---

## Summary

You now have:
- ✅ Headscale server running on OCI free tier
- ✅ Office subnet router advertising your network
- ✅ Users can access entire office network from anywhere
- ✅ Secure mesh VPN with no port forwarding needed
- ✅ Access to ESXi and all VMs via private IPs

**Key Files:**
- Headscale config: `/etc/headscale/config.yaml`
- Headscale database: `/var/lib/headscale/db.sqlite`
- Service logs: `sudo journalctl -u headscale -f`

**Remember:**
- Keep pre-auth keys secure
- Regularly update Headscale and Tailscale
- Monitor access logs
- Back up Headscale database

For support:
- Headscale Docs: https://headscale.net
- Tailscale Docs: https://tailscale.com/kb
