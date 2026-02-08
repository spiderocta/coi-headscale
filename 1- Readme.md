# Headscale VPN Setup - README

Complete solution for accessing your office network (including ESXi and VMs) from home using Headscale on Oracle Cloud Infrastructure (OCI) free tier.

## 📁 Files Included

1. **headscale-setup-guide.md** - Complete detailed documentation
2. **install-headscale.sh** - Automated Headscale installation for OCI
3. **setup-subnet-router.sh** - Office server configuration (Linux)
4. **setup-user-client.sh** - User device configuration (Linux/Mac)
5. **setup-subnet-router.ps1** - Office server configuration (Windows)
6. **setup-user-client.ps1** - User device configuration (Windows)
7. **quick-reference.md** - Command cheat sheet
8. **README.md** - This file

---

## 🚀 Quick Start (3 Simple Steps)

### Step 1: Setup OCI Headscale Server (10 minutes)

1. Create an OCI free tier account at https://www.oracle.com/cloud/free/
2. Create a Ubuntu 22.04 VM (VM.Standard.E2.1.Micro - Always Free)
3. Configure Security List to allow ports: 22, 80, 443, 8080, 3478
4. SSH into the VM:
   ```bash
   ssh ubuntu@<YOUR_OCI_PUBLIC_IP>
   ```
5. Upload and run the installation script:
   ```bash
   wget https://raw.githubusercontent.com/<YOUR_REPO>/install-headscale.sh
   chmod +x install-headscale.sh
   ./install-headscale.sh
   ```
   
   Or copy the `install-headscale.sh` file to the server and run:
   ```bash
   chmod +x install-headscale.sh
   ./install-headscale.sh
   ```

6. Create a user and pre-auth key:
   ```bash
   sudo headscale users create office
   sudo headscale preauthkeys create --user office --reusable --expiration 720h
   ```
   
   **SAVE THE PRE-AUTH KEY!** You'll need it for all clients.

### Step 2: Setup Office Subnet Router (5 minutes)

On your office server (Linux or Windows) that will route traffic:

**Linux:**
```bash
chmod +x setup-subnet-router.sh
./setup-subnet-router.sh
```

**Windows (PowerShell as Administrator):**
```powershell
.\setup-subnet-router.ps1
```

You'll be prompted for:
- Headscale server URL (e.g., `http://123.45.67.89:8080`)
- Office subnet(s) (e.g., `192.168.1.0/24`)
- Pre-auth key from Step 1

Then on your Headscale server, approve the routes:
```bash
sudo headscale nodes list  # Find the NODE_ID
sudo headscale routes enable -i <NODE_ID> -r 192.168.1.0/24
```

### Step 3: Setup User Devices (2 minutes per user)

On each user's device:

**Linux/Mac:**
```bash
chmod +x setup-user-client.sh
./setup-user-client.sh
```

**Windows (PowerShell as Administrator):**
```powershell
.\setup-user-client.ps1
```

You'll be prompted for:
- Headscale server URL
- Pre-auth key from Step 1

**Done!** Users can now access the entire office network.

---

## 📊 What You'll Have

```
User Laptop (Home) ←→ Headscale (OCI Cloud) ←→ Office Router ←→ Office Network
     100.x.x.x              Public IP            100.x.x.y      192.168.1.0/24
                                                                    ↓
                                                            ESXi: 192.168.1.10
                                                            VM1:  192.168.1.20
                                                            VM2:  192.168.1.21
```

---

## 💡 Usage Examples

Once connected, users can access office resources directly:

### Access ESXi Web Interface
```
Open browser: https://192.168.1.10
```

### SSH to Office Server
```bash
ssh admin@192.168.1.100
```

### RDP to Windows VM
```bash
# Linux
rdesktop 192.168.1.50

# Windows
mstsc /v:192.168.1.50
```

### Access Network Shares
```
Windows: \\192.168.1.60\share
Linux:   smb://192.168.1.60/share
```

---

## 🔧 Common Tasks

### Add a New User

1. On Headscale server:
   ```bash
   sudo headscale users create newuser
   sudo headscale preauthkeys create --user newuser --reusable --expiration 720h
   ```

2. Give the key to the new user

3. User runs setup script with the key

### Check Connected Devices

On Headscale server:
```bash
sudo headscale nodes list
```

### View Logs

```bash
# Headscale server
sudo journalctl -u headscale -f

# Client (Linux)
sudo journalctl -u tailscaled -f
```

### Remove a Device

```bash
sudo headscale nodes list  # Find NODE_ID
sudo headscale nodes delete -i <NODE_ID>
```

---

## 🔍 Troubleshooting

### Can't Connect to Headscale

1. Check OCI Security List has ports open (8080, 3478, 443)
2. Check UFW firewall on OCI VM:
   ```bash
   sudo ufw status
   ```
3. Check Headscale is running:
   ```bash
   sudo systemctl status headscale
   ```

### Can't Access Office Network

1. Verify routes are enabled on Headscale:
   ```bash
   sudo headscale routes list
   ```
2. Check IP forwarding on subnet router:
   ```bash
   sysctl net.ipv4.ip_forward
   ```
3. Verify client is accepting routes:
   ```bash
   tailscale status
   ```

### Subnet Router Not Working

Reconnect the subnet router:
```bash
sudo tailscale down
sudo tailscale up --login-server=http://<HEADSCALE_IP>:8080 \
  --advertise-routes=192.168.1.0/24 \
  --accept-routes
```

Then approve routes on Headscale server again.

---

## 📚 Documentation

- **Full Guide**: See `headscale-setup-guide.md` for complete documentation
- **Quick Reference**: See `quick-reference.md` for common commands
- **Headscale Docs**: https://headscale.net
- **Tailscale KB**: https://tailscale.com/kb

---

## 🔒 Security Notes

1. **Production Use**: Enable HTTPS with Let's Encrypt or Caddy
2. **Rotate Keys**: Generate new pre-auth keys regularly
3. **Monitor Access**: Check logs for suspicious activity
4. **Keep Updated**: Update Headscale and Tailscale regularly
5. **Use ACLs**: Restrict access between users/networks if needed

---

## 💰 Cost

**Zero!** Everything runs on OCI's Always Free tier:
- 1x VM.Standard.E2.1.Micro instance
- 1 CPU, 1GB RAM
- More than enough for Headscale

---

## ⚡ Performance

- Direct peer-to-peer connections when possible
- DERP relay for NAT traversal
- Typical latency: 20-50ms (depends on location)
- Bandwidth: Limited by your internet connection

---

## 🆘 Getting Help

1. Check `quick-reference.md` for common commands
2. Review `headscale-setup-guide.md` for detailed explanations
3. Check Headscale logs: `sudo journalctl -u headscale -f`
4. Visit Headscale GitHub: https://github.com/juanfont/headscale

---

## 📝 License

These scripts are provided as-is for educational and personal use.

---

## 🎯 Next Steps

After basic setup:

1. **Enable HTTPS** - Use Caddy or Certbot for production
2. **Setup ACLs** - Control which users can access what
3. **Configure DNS** - Use MagicDNS for easy device names
4. **Backup Database** - Regularly backup `/var/lib/headscale/db.sqlite`
5. **Monitor Resources** - Check OCI VM resources periodically

---

## ✅ Checklist

Setup Progress:

- [ ] OCI account created
- [ ] OCI VM created and configured
- [ ] Headscale installed on OCI
- [ ] User created in Headscale
- [ ] Pre-auth key generated
- [ ] Office subnet router configured
- [ ] Routes approved on Headscale
- [ ] First user device connected
- [ ] Can access office network from home
- [ ] ESXi accessible remotely

---

**Happy networking! You now have a secure, free VPN to access your entire office network from anywhere!** 🎉
