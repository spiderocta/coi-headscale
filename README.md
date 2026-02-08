# coi-headscale

I'll help you set up a Headscale server on Oracle Cloud Infrastructure (OCI) with Tailscale clients for secure remote access to your office network. This will create a mesh VPN where users can access all office resources including ESXi and VMs.

## Overview

**What we're building:**
- Headscale (self-hosted Tailscale control server) on OCI free tier
- Tailscale clients on office server (subnet router)
- Tailscale clients on user devices
- Access to entire office network (private IPs) from anywhere

Let me create a comprehensive guide with ready-to-run scripts.Now let me create the ready-to-run scripts as separate files:Now let me create one final script to make all the bash scripts executable and package everything together:Now let me move all files to the outputs directory so you can access them:Let me try a different approach:Perfect! Now let me present all the files to you:Perfect! I've created a complete Headscale VPN solution for you. Here's what you have:

## 📦 **What's Included:**

### Documentation:
1. **README.md** - Start here! Quick 3-step setup guide
2. **headscale-setup-guide.md** - Complete detailed documentation (10 parts)
3. **quick-reference.md** - Command cheat sheet for daily use

### Ready-to-Run Scripts:

**For OCI Server:**
- `install-headscale.sh` - Automated Headscale installation

**For Office Subnet Router:**
- `setup-subnet-router.sh` (Linux)
- `setup-subnet-router.ps1` (Windows)

**For User Devices:**
- `setup-user-client.sh` (Linux/Mac)
- `setup-user-client.ps1` (Windows)

## 🚀 **Quick Setup (3 Steps):**

1. **OCI Server** → Run `install-headscale.sh` → Get pre-auth key
2. **Office Server** → Run subnet router script → Approve routes
3. **User Devices** → Run client script → Access office network!

## ✨ **Key Features:**

- ✅ **100% Free** - Uses OCI Always Free tier
- ✅ **Full Network Access** - ESXi, VMs, everything
- ✅ **Secure Mesh VPN** - No port forwarding needed
- ✅ **Easy Management** - Simple commands
- ✅ **Cross-Platform** - Works on Linux, Windows, macOS, mobile

All scripts are ready to run with prompts for required information. Just download and execute!
