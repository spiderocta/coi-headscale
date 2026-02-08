#!/bin/bash

##############################################################################
# Headscale Installation Script for OCI Ubuntu
# This script installs and configures Headscale on Oracle Cloud Infrastructure
##############################################################################

set -e

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${GREEN}================================================${NC}"
echo -e "${GREEN}Headscale Installation Script for OCI Ubuntu${NC}"
echo -e "${GREEN}================================================${NC}"
echo ""

# Check if running as root
if [ "$EUID" -eq 0 ]; then 
    echo -e "${RED}Please run this script as a regular user with sudo privileges, not as root${NC}"
    exit 1
fi

# Update system
echo -e "${YELLOW}[1/9] Updating system packages...${NC}"
sudo apt update && sudo apt upgrade -y

# Install required packages
echo -e "${YELLOW}[2/9] Installing dependencies...${NC}"
sudo apt install -y wget curl git sqlite3 apache2-utils ufw

# Configure firewall
echo -e "${YELLOW}[3/9] Configuring UFW firewall...${NC}"
sudo ufw --force enable
sudo ufw allow 22/tcp comment 'SSH'
sudo ufw allow 443/tcp comment 'HTTPS'
sudo ufw allow 8080/tcp comment 'Headscale'
sudo ufw allow 3478/udp comment 'DERP/STUN'
sudo ufw allow 80/tcp comment 'HTTP'
echo -e "${GREEN}Firewall configured${NC}"

# Download latest Headscale
echo -e "${YELLOW}[4/9] Downloading Headscale...${NC}"
HEADSCALE_VERSION="0.23.0"
HEADSCALE_URL="https://github.com/juanfont/headscale/releases/download/v${HEADSCALE_VERSION}/headscale_${HEADSCALE_VERSION}_linux_amd64.deb"

wget -O headscale.deb "${HEADSCALE_URL}"

# Install Headscale
echo -e "${YELLOW}[5/9] Installing Headscale...${NC}"
sudo dpkg -i headscale.deb
rm headscale.deb

# Create necessary directories
echo -e "${YELLOW}[6/9] Creating directories...${NC}"
sudo mkdir -p /etc/headscale
sudo mkdir -p /var/lib/headscale

# Get server address
echo ""
echo -e "${YELLOW}Enter your server address:${NC}"
echo "  - If you have a domain: headscale.yourdomain.com"
echo "  - If using IP only: $(curl -s ifconfig.me)"
read -p "Server address: " SERVER_ADDRESS

if [ -z "$SERVER_ADDRESS" ]; then
    SERVER_ADDRESS=$(curl -s ifconfig.me)
    echo -e "${GREEN}Using public IP: ${SERVER_ADDRESS}${NC}"
fi

# Create configuration file
echo -e "${YELLOW}[7/9] Creating configuration...${NC}"
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
echo -e "${YELLOW}[8/9] Setting permissions...${NC}"
sudo chown -R headscale:headscale /etc/headscale
sudo chown -R headscale:headscale /var/lib/headscale

# Enable and start Headscale
echo -e "${YELLOW}[9/9] Starting Headscale service...${NC}"
sudo systemctl enable headscale
sudo systemctl start headscale

# Wait for service to start
sleep 3

# Check status
echo ""
echo -e "${GREEN}================================================${NC}"
echo -e "${GREEN}Installation Complete!${NC}"
echo -e "${GREEN}================================================${NC}"
echo ""

if sudo systemctl is-active --quiet headscale; then
    echo -e "${GREEN}✓ Headscale is running${NC}"
else
    echo -e "${RED}✗ Headscale failed to start${NC}"
    echo "Check logs with: sudo journalctl -u headscale -n 50"
    exit 1
fi

echo ""
echo -e "${GREEN}Headscale Server URL:${NC} http://${SERVER_ADDRESS}:8080"
echo ""
echo -e "${YELLOW}Next Steps:${NC}"
echo ""
echo "1. Create a user:"
echo "   ${GREEN}sudo headscale users create office${NC}"
echo ""
echo "2. Generate a pre-auth key:"
echo "   ${GREEN}sudo headscale preauthkeys create --user office --reusable --expiration 720h${NC}"
echo ""
echo "3. Use this key on all your Tailscale clients"
echo ""
echo "For more commands, see the documentation."
echo ""
echo -e "${YELLOW}Useful Commands:${NC}"
echo "  View status:  ${GREEN}sudo systemctl status headscale${NC}"
echo "  View logs:    ${GREEN}sudo journalctl -u headscale -f${NC}"
echo "  List nodes:   ${GREEN}sudo headscale nodes list${NC}"
echo "  List users:   ${GREEN}sudo headscale users list${NC}"
echo ""
