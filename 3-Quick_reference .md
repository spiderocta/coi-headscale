# Headscale VPN - Quick Reference Guide

## Installation Order

1. **OCI Server** → Install Headscale
2. **Office Server** → Install Tailscale (Subnet Router)
3. **User Devices** → Install Tailscale (Clients)

---

## Quick Commands Reference

### On Headscale Server (OCI)

```bash
# Create user
sudo headscale users create office

# Generate pre-auth key (reusable, 30 days)
sudo headscale preauthkeys create --user office --reusable --expiration 720h

# List all nodes
sudo headscale nodes list

# List routes
sudo headscale routes list

# Enable routes for a node
sudo headscale routes enable -i <NODE_ID> -r 192.168.1.0/24

# Delete a node
sudo headscale nodes delete -i <NODE_ID>

# View logs
sudo journalctl -u headscale -f

# Check service status
sudo systemctl status headscale

# Restart service
sudo systemctl restart headscale
```

### On Subnet Router (Office Server)

**Linux:**
```bash
# Connect to Headscale and advertise routes
sudo tailscale up --login-server=http://<HEADSCALE_IP>:8080 \
  --advertise-routes=192.168.1.0/24,192.168.2.0/24 \
  --accept-routes \
  --authkey=<PREAUTH_KEY>

# Check status
tailscale status

# Check IP
tailscale ip
```

**Windows (PowerShell as Admin):**
```powershell
tailscale up --login-server="http://<HEADSCALE_IP>:8080" --advertise-routes="192.168.1.0/24" --accept-routes --authkey="<PREAUTH_KEY>"
```

### On User Devices (Clients)

**Linux/macOS:**
```bash
# Connect to Headscale
sudo tailscale up --login-server=http://<HEADSCALE_IP>:8080 \
  --accept-routes \
  --authkey=<PREAUTH_KEY>

# Check status
tailscale status

# Ping another node
tailscale ping <node-name>
```

**Windows (PowerShell as Admin):**
```powershell
tailscale up --login-server="http://<HEADSCALE_IP>:8080" --accept-routes --authkey="<PREAUTH_KEY>"
```

---

## Common Workflows

### Add New User

1. On Headscale server:
```bash
sudo headscale users create <username>
sudo headscale preauthkeys create --user <username> --reusable --expiration 720h
```

2. Give the pre-auth key to the user

3. User runs on their device:
```bash
sudo tailscale up --login-server=http://<HEADSCALE_IP>:8080 --accept-routes --authkey=<KEY>
```

### Add New Subnet to Router

1. On subnet router, reconnect with additional routes:
```bash
sudo tailscale up --login-server=http://<HEADSCALE_IP>:8080 \
  --advertise-routes=192.168.1.0/24,192.168.2.0/24,10.0.0.0/24 \
  --accept-routes
```

2. On Headscale server, enable the new route:
```bash
sudo headscale nodes list  # Find NODE_ID
sudo headscale routes enable -i <NODE_ID> -r 10.0.0.0/24
```

### Remove a Device

On Headscale server:
```bash
sudo headscale nodes list  # Find NODE_ID
sudo headscale nodes delete -i <NODE_ID>
```

---

## Troubleshooting

### Cannot Connect to Headscale

```bash
# Check if Headscale is running
sudo systemctl status headscale

# Check logs
sudo journalctl -u headscale -n 100

# Check firewall
sudo ufw status

# Test connectivity
curl http://<HEADSCALE_IP>:8080/health
```

### Cannot Access Office Network

```bash
# On Headscale server - check routes
sudo headscale routes list
# Ensure routes are "enabled: true"

# On subnet router - check IP forwarding
sysctl net.ipv4.ip_forward
# Should return: net.ipv4.ip_forward = 1

# On client - verify route acceptance
tailscale status
# Look for "accept-routes: true"

# Test connectivity
ping <office-ip>
traceroute <office-ip>
```

### Subnet Router Not Advertising Routes

```bash
# Reconnect with correct flags
sudo tailscale down
sudo tailscale up --login-server=http://<HEADSCALE_IP>:8080 \
  --advertise-routes=192.168.1.0/24 \
  --accept-routes \
  --authkey=<PREAUTH_KEY>

# On Headscale server
sudo headscale nodes list
sudo headscale routes enable -i <NODE_ID> -r 192.168.1.0/24
```

---

## Network Configuration Examples

### Example 1: Single Office Network
```
Office Network: 192.168.1.0/24
ESXi Server: 192.168.1.10
VMs: 192.168.1.20-100

Advertise routes: --advertise-routes=192.168.1.0/24
```

### Example 2: Multiple Office Networks
```
Office LAN: 192.168.1.0/24
Server VLAN: 192.168.2.0/24
Guest Network: 192.168.3.0/24

Advertise routes: --advertise-routes=192.168.1.0/24,192.168.2.0/24,192.168.3.0/24
```

### Example 3: Complex Setup
```
Main Office: 10.0.0.0/16
Branch Office: 172.16.0.0/16
DMZ: 192.168.100.0/24

Advertise routes: --advertise-routes=10.0.0.0/16,172.16.0.0/16,192.168.100.0/24
```

---

## Security Tips

1. **Use HTTPS** for production (use Caddy or Let's Encrypt)
2. **Rotate pre-auth keys** regularly
3. **Monitor access logs** on Headscale server
4. **Use ACLs** to restrict access between users/networks
5. **Keep software updated**:
   ```bash
   # Update Headscale
   wget https://github.com/juanfont/headscale/releases/download/v<VERSION>/headscale_<VERSION>_linux_amd64.deb
   sudo dpkg -i headscale_<VERSION>_linux_amd64.deb
   sudo systemctl restart headscale
   
   # Update Tailscale
   sudo tailscale update
   ```

---

## File Locations

### Headscale Server (OCI)
```
Config:    /etc/headscale/config.yaml
Database:  /var/lib/headscale/db.sqlite
Logs:      sudo journalctl -u headscale
Socket:    /var/run/headscale/headscale.sock
```

### Tailscale Clients

**Linux:**
```
Config:    /var/lib/tailscale/
Logs:      sudo journalctl -u tailscaled
```

**Windows:**
```
Install:   C:\Program Files\Tailscale\
Logs:      Event Viewer → Applications → Tailscale
```

---

## Performance Tips

1. **Use DERP relay** (already enabled in config)
2. **Direct connections** are automatically negotiated
3. **Monitor bandwidth** on subnet router
4. **Use subnet routing** instead of individual device routing
5. **Enable MagicDNS** for easy device naming

---

## Access Examples After Setup

Once everything is configured, users can:

```bash
# SSH to office server
ssh admin@192.168.1.100

# Access ESXi web interface
# Open browser: https://192.168.1.10

# RDP to Windows server
mstsc /v:192.168.1.50

# Access network shares (Windows)
\\192.168.1.60\share

# Access network shares (Linux)
smb://192.168.1.60/share

# Ping office gateway
ping 192.168.1.1
```

---

## Emergency Procedures

### If Headscale Server Goes Down

Users will lose connectivity. To restore:

```bash
# SSH to OCI VM
ssh ubuntu@<OCI_IP>

# Check and restart service
sudo systemctl status headscale
sudo systemctl restart headscale

# Check logs for errors
sudo journalctl -u headscale -n 100 --no-pager
```

### If Subnet Router Goes Down

Users can't access office network. Restore by:

1. Restart the subnet router machine
2. Verify Tailscale is running:
   ```bash
   sudo systemctl status tailscaled
   ```
3. Reconnect if needed:
   ```bash
   sudo tailscale up --login-server=http://<HEADSCALE_IP>:8080 \
     --advertise-routes=192.168.1.0/24 --accept-routes
   ```

---

## Backup & Recovery

### Backup Headscale Database

```bash
# On Headscale server
sudo systemctl stop headscale
sudo cp /var/lib/headscale/db.sqlite /var/lib/headscale/db.sqlite.backup
sudo systemctl start headscale

# Download to local machine
scp ubuntu@<OCI_IP>:/var/lib/headscale/db.sqlite.backup ./
```

### Restore Database

```bash
# Upload backup
scp db.sqlite.backup ubuntu@<OCI_IP>:/tmp/

# On Headscale server
sudo systemctl stop headscale
sudo cp /tmp/db.sqlite.backup /var/lib/headscale/db.sqlite
sudo chown headscale:headscale /var/lib/headscale/db.sqlite
sudo systemctl start headscale
```

---

## Support & Resources

- Headscale Docs: https://headscale.net
- Tailscale KB: https://tailscale.com/kb
- OCI Free Tier: https://www.oracle.com/cloud/free/
- Community Forum: https://github.com/juanfont/headscale/discussions
