# TrueNAS Scale Debian Container Setup Guide

**Date Created:** 2025-11-21 05:57:20 UTC  
**Author:** njayasathya  
**Purpose:** Complete guide for setting up Debian container with SSH access and static IP in TrueNAS Scale

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Step 1: Creating a Debian Container](#step-1-creating-a-debian-container)
4. [Step 2: Configuring Static IP Address](#step-2-configuring-static-ip-address)
5. [Step 3: Installing SSH Server](#step-3-installing-ssh-server)
6. [Step 4: Creating SSH User](#step-4-creating-ssh-user)
7. [Step 5: Setting Up SSH Keys](#step-5-setting-up-ssh-keys)
8. [Step 6: Connecting via VSCode Remote SSH](#step-6-connecting-via-vscode-remote-ssh)
9. [Troubleshooting](#troubleshooting)
10. [Security Recommendations](#security-recommendations)

---

## Overview

This guide walks through the complete process of:
- Installing a Debian container in TrueNAS Scale
- Configuring a static IP address using systemd-networkd
- Setting up SSH server for remote access
- Creating a dedicated SSH user
- Configuring SSH key-based authentication
- Connecting via VSCode Remote SSH extension

---

## Prerequisites

- **TrueNAS Scale** installation with Apps (Kubernetes) enabled
- **SSH key generation tools** on your local machine (Windows, macOS, or Linux)
- **VSCode** with Remote SSH extension (optional but recommended)
- Network connectivity between your machine and TrueNAS Scale
- Basic command-line familiarity

---

## Step 1: Creating a Debian Container

### Via TrueNAS Scale Web UI

1. Navigate to **Apps** in the TrueNAS Scale web interface
2. Click **Discover Apps** or **Launch Docker Image**
3. Search for `debian` or `debian:latest`
4. Click **Install** and configure:
   - **Name:** `wireguard` (or your preferred name)
   - **Image:** `debian:latest`
   - **Command:** `sleep infinity` (keeps container running)
5. Configure storage and networking as needed
6. Click **Install**

### Access Container Shell

1. Go to **Apps** → Your Debian container
2. Click **Shell** to open a terminal session

---

## Step 2: Configuring Static IP Address

### Check Current Network Configuration

Inside the container:

```bash
# View current IP and network info
ip addr show
ip route show

# Check network configuration directory
ls -la /etc/systemd/network/
```

Expected output should show:
- Current IP (usually assigned via DHCP)
- Default gateway
- Active network interface (e.g., `eth0`)

### Edit Network Configuration

The container uses **systemd-networkd** for network management. Edit the network config:

```bash
cat > /etc/systemd/network/eth0.network << 'EOF'
[Match]
Name=eth0

[Network]
Address=192.168.1.100/24
Gateway=192.168.1.254
DNS=8.8.8.8 8.8.4.4
EOF
```

**Replace with your values:**
- `192.168.1.100` → Your desired static IP
- `192.168.1.254` → Your gateway/router IP
- `8.8.8.8 8.8.4.4` → Your DNS servers

### Apply Network Changes

```bash
# Restart networking service
systemctl restart systemd-networkd

# Verify new IP was assigned
ip addr show eth0
```

---

## Step 3: Installing SSH Server

### Install SSH Package

```bash
# Update package manager
apt update

# Install OpenSSH server
apt install -y openssh-server
```

### Start SSH Service

```bash
# Start SSH immediately
systemctl start ssh

# Enable SSH to start on boot
systemctl enable ssh

# Verify SSH is running
systemctl status ssh
```

### Verify SSH is Listening

```bash
# Check if SSH is listening on port 22
ss -tlnp | grep ssh

# Or using netstat
netstat -tlnp | grep ssh
```

Expected output should show SSH listening on `0.0.0.0:22`

---

## Step 4: Creating SSH User

### Create New User

```bash
# Create user with home directory and bash shell
useradd -m -s /bin/bash njayasathya

# Verify user was created
id njayasathya
```

### Add User to Sudo Group (Optional)

```bash
# Allow user to run sudo commands
usermod -aG sudo njayasathya
```

### Create SSH Directory

```bash
# Create .ssh directory for the user
mkdir -p /home/njayasathya/.ssh
chmod 700 /home/njayasathya/.ssh

# Verify permissions
ls -la /home/njayasathya/.ssh
```

---

## Step 5: Setting Up SSH Keys

### On Your Local Machine (Windows, macOS, or Linux)

#### Generate SSH Key Pair

**Windows PowerShell:**

```powershell
# Create .ssh directory if it doesn't exist
mkdir $HOME\.ssh

# Generate ED25519 key (recommended)
ssh-keygen -t ed25519 -f $HOME\.ssh\truenas_wireguard_debian -C "njayasathya@wireguard" -N ""

# Or generate RSA key (4096-bit)
ssh-keygen -t rsa -b 4096 -f $HOME\.ssh\truenas_wireguard_debian -C "njayasathya@wireguard" -N ""
```

**macOS/Linux:**

```bash
# Create .ssh directory if it doesn't exist
mkdir -p ~/.ssh

# Generate ED25519 key
ssh-keygen -t ed25519 -f ~/.ssh/truenas_wireguard_debian -C "njayasathya@wireguard" -N ""
```

#### View Your Public Key

**Windows PowerShell:**

```powershell
cat $HOME\.ssh\truenas_wireguard_debian.pub
```

**macOS/Linux:**

```bash
cat ~/.ssh/truenas_wireguard_debian.pub
```

Copy the entire output (should start with `ssh-ed25519` or `ssh-rsa`)

### On Your Container

#### Add Public Key to Container

Inside your container, add your public key:

```bash
# Create authorized_keys file
cat >> /home/njayasathya/.ssh/authorized_keys << 'EOF'
<PASTE-YOUR-PUBLIC-KEY-HERE>
EOF
```

Replace `<PASTE-YOUR-PUBLIC-KEY-HERE>` with the actual key from your local machine.

#### Set Correct Permissions

```bash
# Fix file permissions
chmod 600 /home/njayasathya/.ssh/authorized_keys

# Fix directory ownership
chown -R njayasathya:njayasathya /home/njayasathya/.ssh

# Verify permissions
ls -la /home/njayasathya/.ssh/
```

### Configure SSH on Local Machine

#### Create SSH Config File

**Windows PowerShell:**

```powershell
# Edit or create SSH config
notepad $HOME\.ssh\config
```

**macOS/Linux:**

```bash
nano ~/.ssh/config
```

Add the following:

```
Host wireguard
  HostName 192.168.1.100
  User njayasathya
  IdentityFile ~/.ssh/truenas_wireguard_debian
  AddKeysToAgent yes
  IdentitiesOnly yes
```

**Windows users:** Replace `~` with `$HOME` in the path if needed.

#### Test SSH Connection

**Windows PowerShell:**

```powershell
ssh wireguard
```

**macOS/Linux:**

```bash
ssh wireguard
```

You should connect without entering a password!

---

## Step 6: Connecting via VSCode Remote SSH

### Install Remote SSH Extension

1. Open **VSCode**
2. Go to **Extensions** (Ctrl+Shift+X / Cmd+Shift+X)
3. Search for `Remote - SSH`
4. Install the official Microsoft extension

### Configure Connection

1. Press **Ctrl+Shift+P** (Cmd+Shift+P on macOS)
2. Type `Remote-SSH: Open SSH Configuration File`
3. Select your SSH config file (`~/.ssh/config`)
4. Verify the `wireguard` host is present
5. Save and close the file

### Connect to Container

1. Press **Ctrl+Shift+P**
2. Type `Remote-SSH: Connect to Host`
3. Select `wireguard` from the list
4. VSCode will open a new window connected to your container
5. Trust the host if prompted

### You're Connected!

Once connected, you can:
- Open folders and browse files
- Edit files with full VSCode features
- Open an integrated terminal
- Install VSCode extensions on the remote

---

## Troubleshooting

### SSH Connection Refused

**Error:** `ssh: connect to host 192.168.1.100 port 22: Connection refused`

**Solution:**

```bash
# Verify SSH is running
systemctl status ssh

# Check if SSH is listening
ss -tlnp | grep ssh

# If not running, start it
systemctl start ssh
systemctl enable ssh
```

### Connection Timeout

**Error:** Connection times out

**Solution:**

1. Verify the container's IP address:
   ```bash
   ip addr show
   ```

2. Test connectivity from your local machine:
   ```bash
   ping 192.168.1.100
   ```

3. Check firewall settings (unlikely in container, but verify)

### Permission Denied (Public Key)

**Error:** `Permission denied (publickey)`

**Solution:**

1. Verify SSH key was added:
   ```bash
   cat /home/njayasathya/.ssh/authorized_keys
   ```

2. Check file permissions:
   ```bash
   ls -la /home/njayasathya/.ssh/
   ```
   
   Should show:
   ```
   drwx------  .ssh
   -rw-------  authorized_keys
   ```

3. Fix permissions if needed:
   ```bash
   chmod 700 /home/njayasathya/.ssh
   chmod 600 /home/njayasathya/.ssh/authorized_keys
   chown -R njayasathya:njayasathya /home/njayasathya/.ssh
   ```

### Network Configuration Not Applied

**Error:** Static IP not assigned

**Solution:**

1. Check the network config file:
   ```bash
   cat /etc/systemd/network/eth0.network
   ```

2. Restart networking:
   ```bash
   systemctl restart systemd-networkd
   ```

3. Wait a few seconds and check IP:
   ```bash
   ip addr show eth0
   ```

### SSH Service Not Found

**Error:** `Unit ssh.service not found`

**Solution:**

```bash
# Install SSH server
apt update && apt install -y openssh-server

# Start service
systemctl start ssh
systemctl enable ssh
```

---

## Security Recommendations

### 1. Disable Root SSH Login

Edit SSH configuration:

```bash
sudo nano /etc/ssh/sshd_config
```

Find or add:

```
PermitRootLogin no
```

Or use sed:

```bash
sudo sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

### 2. Disable Password Authentication

```bash
# Edit SSH config
sudo nano /etc/ssh/sshd_config

# Find and change:
# PasswordAuthentication yes
# to
# PasswordAuthentication no
```

Or use sed:

```bash
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

### 3. Change Default SSH Port (Optional)

```bash
# Edit SSH config
sudo nano /etc/ssh/sshd_config

# Find and change:
# Port 22
# to
# Port 2222
```

Then update your SSH config on local machine:

```
Host wireguard
  HostName 192.168.1.100
  User njayasathya
  Port 2222
  IdentityFile ~/.ssh/truenas_wireguard_debian
```

### 4. Restrict SSH to Specific IPs (If Applicable)

```bash
sudo nano /etc/ssh/sshd_config

# Add:
AllowUsers njayasathya
```

### 5. Keep System Updated

```bash
# Update package manager and packages regularly
sudo apt update && sudo apt upgrade -y
```

---

## Quick Reference Commands

### On Container

```bash
# Check IP address
ip addr show

# Check SSH status
systemctl status ssh

# View SSH config
cat /etc/ssh/sshd_config

# Restart SSH after config changes
systemctl restart ssh

# Check SSH is listening
ss -tlnp | grep ssh

# View network config
cat /etc/systemd/network/eth0.network

# Restart network
systemctl restart systemd-networkd
```

### On Local Machine

```bash
# Connect via SSH
ssh wireguard

# Test SSH connection
ssh -v wireguard

# Copy file to container
scp myfile njayasathya@192.168.1.100:~/

# Copy file from container
scp njayasathya@192.168.1.100:~/myfile .

# Run command on container
ssh wireguard "ls -la"
```

---

## Additional Resources

- [TrueNAS Documentation](https://www.truenas.com/docs/)
- [OpenSSH Manual](https://man.openbsd.org/ssh)
- [VSCode Remote SSH Documentation](https://code.visualstudio.com/docs/remote/ssh)
- [systemd-networkd Manual](https://www.freedesktop.org/software/systemd/man/latest/systemd.network.html)

---

## Support & Feedback

For issues or questions:
1. Check the **Troubleshooting** section above
2. Review system logs: `journalctl -u ssh -n 20`
3. Test manually before using VSCode
4. Verify all permissions are correctly set

---

**Last Updated:** 2025-11-21 05:57:20 UTC  
**Version:** 1.0
