#!/bin/bash

##############################################################################
# Tailscale User Client Setup Script
# This installs and configures Tailscale on user devices (Linux/Mac)
##############################################################################

set -e

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${GREEN}================================================${NC}"
echo -e "${GREEN}Tailscale User Client Setup${NC}"
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

read -p "Pre-auth key from Headscale: " PREAUTH_KEY
if [ -z "$PREAUTH_KEY" ]; then
    echo -e "${RED}Error: Pre-auth key is required${NC}"
    exit 1
fi

echo ""
echo -e "${YELLOW}[1/2] Installing Tailscale...${NC}"

# Detect OS
if [[ "$OSTYPE" == "linux-gnu"* ]]; then
    # Linux
    if command -v apt-get &> /dev/null; then
        # Debian/Ubuntu
        curl -fsSL https://tailscale.com/install.sh | sh
    elif command -v yum &> /dev/null; then
        # RedHat/CentOS
        curl -fsSL https://tailscale.com/install.sh | sh
    else
        echo -e "${RED}Unsupported Linux distribution${NC}"
        exit 1
    fi
elif [[ "$OSTYPE" == "darwin"* ]]; then
    # macOS
    if ! command -v tailscale &> /dev/null; then
        echo -e "${YELLOW}Please install Tailscale for macOS from:${NC}"
        echo "https://tailscale.com/download/mac"
        echo ""
        read -p "Press Enter after installing Tailscale..."
    fi
else
    echo -e "${RED}Unsupported operating system: $OSTYPE${NC}"
    exit 1
fi

echo -e "${GREEN}✓ Tailscale installed${NC}"

echo ""
echo -e "${YELLOW}[2/2] Connecting to Headscale...${NC}"

# For macOS, use the full path
if [[ "$OSTYPE" == "darwin"* ]]; then
    if [ -f "/Applications/Tailscale.app/Contents/MacOS/Tailscale" ]; then
        sudo /Applications/Tailscale.app/Contents/MacOS/Tailscale up \
          --login-server="${HEADSCALE_URL}" \
          --accept-routes \
          --authkey="${PREAUTH_KEY}"
    else
        # Fallback to regular command
        sudo tailscale up \
          --login-server="${HEADSCALE_URL}" \
          --accept-routes \
          --authkey="${PREAUTH_KEY}"
    fi
else
    # Linux
    sudo tailscale up \
      --login-server="${HEADSCALE_URL}" \
      --accept-routes \
      --authkey="${PREAUTH_KEY}"
fi

echo ""
echo -e "${GREEN}================================================${NC}"
echo -e "${GREEN}Client Setup Complete!${NC}"
echo -e "${GREEN}================================================${NC}"
echo ""

# Get connection info
sleep 2
TAILSCALE_IP=$(tailscale ip -4 2>/dev/null || echo "N/A")

echo -e "${GREEN}Tailscale IP:${NC} ${TAILSCALE_IP}"
echo ""
echo -e "${GREEN}You are now connected to your office network!${NC}"
echo ""
echo -e "${YELLOW}What you can do now:${NC}"
echo "  • Access office servers by their private IPs"
echo "  • Connect to ESXi: https://<esxi-ip>"
echo "  • SSH to office machines: ssh user@<office-ip>"
echo "  • Access internal web services"
echo ""
echo -e "${YELLOW}Useful Commands:${NC}"
echo "  Check status:  ${GREEN}tailscale status${NC}"
echo "  Show IP:       ${GREEN}tailscale ip${NC}"
echo "  Ping node:     ${GREEN}tailscale ping <node-name>${NC}"
echo "  Disconnect:    ${GREEN}sudo tailscale down${NC}"
echo "  Reconnect:     ${GREEN}sudo tailscale up --login-server=${HEADSCALE_URL} --accept-routes${NC}"
echo ""
