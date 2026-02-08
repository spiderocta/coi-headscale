#!/bin/bash

##############################################################################
# Office Subnet Router Setup Script
# This configures a server to act as a subnet router for your office network
##############################################################################

set -e

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${GREEN}================================================${NC}"
echo -e "${GREEN}Office Subnet Router Setup${NC}"
echo -e "${GREEN}================================================${NC}"
echo ""

# Get configuration
echo -e "${YELLOW}Please provide the following information:${NC}"
echo ""

read -p "Headscale server URL (e.g., http://1.2.3.4:8080): " HEADSCALE_URL
if [ -z "$HEADSCALE_URL" ]; then
    echo -e "${RED}Error: Headscale URL is required${NC}"
    exit 1
fi

echo ""
echo "Enter your office subnet(s):"
echo "  Single subnet:   192.168.1.0/24"
echo "  Multiple subnets: 192.168.1.0/24,192.168.2.0/24,10.0.0.0/24"
read -p "Office subnet(s): " OFFICE_SUBNET
if [ -z "$OFFICE_SUBNET" ]; then
    echo -e "${RED}Error: Office subnet is required${NC}"
    exit 1
fi

read -p "Pre-auth key from Headscale: " PREAUTH_KEY
if [ -z "$PREAUTH_KEY" ]; then
    echo -e "${RED}Error: Pre-auth key is required${NC}"
    exit 1
fi

echo ""
echo -e "${YELLOW}[1/4] Installing Tailscale...${NC}"
curl -fsSL https://tailscale.com/install.sh | sh

echo ""
echo -e "${YELLOW}[2/4] Enabling IP forwarding...${NC}"

# Enable IP forwarding permanently
if ! grep -q "net.ipv4.ip_forward = 1" /etc/sysctl.conf; then
    echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
fi

if ! grep -q "net.ipv6.conf.all.forwarding = 1" /etc/sysctl.conf; then
    echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.conf
fi

sudo sysctl -p
echo -e "${GREEN}✓ IP forwarding enabled${NC}"

echo ""
echo -e "${YELLOW}[3/4] Configuring firewall...${NC}"

# Check if UFW is installed and active
if command -v ufw &> /dev/null; then
    sudo ufw allow 41641/udp comment 'Tailscale'
    echo -e "${GREEN}✓ Firewall configured${NC}"
fi

echo ""
echo -e "${YELLOW}[4/4] Connecting to Headscale...${NC}"
sudo tailscale up --login-server="${HEADSCALE_URL}" \
  --advertise-routes="${OFFICE_SUBNET}" \
  --accept-routes \
  --authkey="${PREAUTH_KEY}" \
  --advertise-exit-node

echo ""
echo -e "${GREEN}================================================${NC}"
echo -e "${GREEN}Subnet Router Setup Complete!${NC}"
echo -e "${GREEN}================================================${NC}"
echo ""

# Get node info
sleep 2
TAILSCALE_IP=$(tailscale ip -4)
TAILSCALE_HOSTNAME=$(hostname)

echo -e "${GREEN}Tailscale IP:${NC} ${TAILSCALE_IP}"
echo -e "${GREEN}Hostname:${NC} ${TAILSCALE_HOSTNAME}"
echo ""
echo -e "${YELLOW}IMPORTANT: Complete these steps on your Headscale server:${NC}"
echo ""
echo "1. List nodes to find this router's ID:"
echo "   ${GREEN}sudo headscale nodes list${NC}"
echo ""
echo "2. Enable the advertised routes (look for node with IP ${TAILSCALE_IP}):"
echo "   ${GREEN}sudo headscale routes enable -i <NODE_ID> -r ${OFFICE_SUBNET}${NC}"
echo ""
echo "3. Verify routes are enabled:"
echo "   ${GREEN}sudo headscale routes list${NC}"
echo ""
echo -e "${YELLOW}Testing:${NC}"
echo "  Check status:  ${GREEN}tailscale status${NC}"
echo "  Show IP:       ${GREEN}tailscale ip${NC}"
echo "  View logs:     ${GREEN}sudo journalctl -u tailscaled -f${NC}"
echo ""
