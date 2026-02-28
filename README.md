# 🔒 Linux Security Homelab: System Hardening, Firewall Configuration & SSL Implementation

> A multi-VM virtualized lab environment built to practice and document real-world Linux system hardening, firewall management, and encrypted communications.

---

## 📋 Project Overview

This lab simulates a secure networked environment using multiple Linux virtual machines. The goal was to apply enterprise-grade hardening techniques from scratch — configuring firewalls, enforcing SSH key authentication, implementing SSL/TLS encryption, and validating every control through hands-on testing.

**Skills demonstrated:** Linux system administration · UFW firewall · SSH hardening · OpenSSL · SSL/TLS · Network interface configuration · Service management · User privilege control

---

## 🏗️ Lab Architecture

```
┌──────────────────────────────────────────────────┐
│                 VirtualBox Host                  │
│                                                  │
│  ┌─────────────────┐     ┌─────────────────────┐ │
│  │  Kali Linux VM  │     │   Linux Mint VM      │ │
│  │  (Pen Testing / │     │   (Workstation /     │ │
│  │   Validation)   │     │    Admin Client)     │ │
│  │                 │     │                      │ │
│  │  192.168.x.10   │     │   192.168.x.30       │ │
│  └────────┬────────┘     └──────────┬───────────┘ │
│           │      Internal Network   │             │
│           └────────────┬────────────┘             │
│                        │                         │
│           ┌────────────┴────────────┐             │
│           │    Ubuntu Server VM     │             │
│           │    (Hardened Target)    │             │
│           │                        │             │
│           │  - UFW Firewall         │             │
│           │  - SSH (key-auth only)  │             │
│           │  - HTTPS / OpenSSL      │             │
│           │  - Apache2 web server   │             │
│           │                        │             │
│           │    192.168.x.20        │             │
│           └────────────────────────┘             │
└──────────────────────────────────────────────────┘
```

**Network type:** VirtualBox Internal Network (isolated)  
**OS versions:** Ubuntu Server 22.04 LTS · Linux Mint 21 · Kali Linux (rolling)

---

## ⚙️ Setup & Configuration

### 1. VirtualBox Network Setup

Each VM uses an **Internal Network** adapter so they can communicate with each other but remain isolated from the internet during testing.

```bash
# Assign static IPs on Ubuntu Server
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses:
        - 192.168.x.20/24
      gateway4: 192.168.x.1
      nameservers:
        addresses: [8.8.8.8]
```

```bash
sudo netplan apply
ip a  # Verify interface assignment
```

---

### 2. UFW Firewall Configuration

The principle here is **default deny** — block everything, then explicitly allow only what's needed.

```bash
# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow only essential ports
sudo ufw allow 22/tcp    # SSH — remote administration
sudo ufw allow 443/tcp   # HTTPS — encrypted web traffic
sudo ufw allow 80/tcp    # HTTP — redirect to HTTPS

# Enable and verify
sudo ufw enable
sudo ufw status verbose
```

**Expected output:**
```
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
```

#### Additional hardening rules applied:
```bash
# Limit SSH brute-force attempts (max 6 connections per 30 seconds)
sudo ufw limit 22/tcp

# Block a specific IP (example — used during pen test validation)
sudo ufw deny from 192.168.x.10 to any port 3306

# Logging enabled for auditing
sudo ufw logging on
sudo tail -f /var/log/ufw.log
```

---

### 3. SSH Hardening

Default SSH configuration has several weaknesses. These changes close the most common attack vectors.

#### Generate SSH Key Pair (on client machine)
```bash
ssh-keygen -t ed25519 -C "daniel-homelab" -f ~/.ssh/homelab_key
# Private key: ~/.ssh/homelab_key  (never share this)
# Public key:  ~/.ssh/homelab_key.pub  (copied to server)
```

#### Copy Public Key to Server
```bash
ssh-copy-id -i ~/.ssh/homelab_key.pub daniel@192.168.x.20
# Or manually append to ~/.ssh/authorized_keys on the server
```

#### Harden `/etc/ssh/sshd_config`
```bash
sudo nano /etc/ssh/sshd_config
```

```ini
# Disable password authentication entirely
PasswordAuthentication no
ChallengeResponseAuthentication no

# Disable root login over SSH
PermitRootLogin no

# Change default port (obscurity layer — not security by itself)
Port 2222

# Only allow specific users
AllowUsers daniel

# Idle timeout — disconnect inactive sessions after 5 minutes
ClientAliveInterval 300
ClientAliveCountMax 0

# Disable X11 forwarding (not needed)
X11Forwarding no

# Use modern key exchange algorithms only
KexAlgorithms curve25519-sha256,diffie-hellman-group14-sha256
```

```bash
# Restart SSH and verify
sudo systemctl restart sshd
sudo systemctl status sshd

# Test key-based login from client
ssh -i ~/.ssh/homelab_key -p 2222 daniel@192.168.x.20

# Verify password login is rejected
ssh -o PubkeyAuthentication=no daniel@192.168.x.20
# Expected: "Permission denied (publickey)"
```

---

### 4. SSL/TLS with OpenSSL (Self-Signed Certificate)

Enforcing HTTPS ensures all traffic to the server is encrypted in transit — even in a lab environment.

#### Generate Certificate
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/homelab.key \
  -out /etc/ssl/certs/homelab.crt \
  -subj "/CN=homelab.local/O=DanielLab/C=US"
```

**Flags explained:**
| Flag | Meaning |
|---|---|
| `-x509` | Output a self-signed certificate (not a CSR) |
| `-nodes` | No passphrase on the private key |
| `-days 365` | Certificate valid for 1 year |
| `-newkey rsa:2048` | Generate a new 2048-bit RSA key pair |
| `-keyout` | Path to save the private key |
| `-out` | Path to save the certificate |

#### Configure Apache2 for HTTPS
```bash
sudo a2enmod ssl
sudo nano /etc/apache2/sites-available/default-ssl.conf
```

```apache
<VirtualHost *:443>
    ServerName homelab.local
    DocumentRoot /var/www/html

    SSLEngine on
    SSLCertificateFile      /etc/ssl/certs/homelab.crt
    SSLCertificateKeyFile   /etc/ssl/private/homelab.key

    # Force modern TLS only
    SSLProtocol             all -SSLv3 -TLSv1 -TLSv1.1
    SSLCipherSuite          HIGH:!aNULL:!MD5
</VirtualHost>
```

```bash
# Redirect HTTP to HTTPS
sudo nano /etc/apache2/sites-available/000-default.conf
```
```apache
<VirtualHost *:80>
    ServerName homelab.local
    Redirect permanent / https://homelab.local/
</VirtualHost>
```

```bash
sudo a2ensite default-ssl
sudo systemctl restart apache2

# Verify certificate details
openssl s_client -connect homelab.local:443 -showcerts
```

---

### 5. Service Hardening & User Privilege Control

Reducing the attack surface means disabling anything that doesn't need to be running.

```bash
# List all running services
sudo systemctl list-units --type=service --state=running

# Disable unnecessary services (examples)
sudo systemctl disable --now cups        # Printing — not needed on a server
sudo systemctl disable --now avahi-daemon # Network discovery — not needed
sudo systemctl disable --now bluetooth

# Verify only intended services remain
sudo ss -tulnp  # Show all listening ports and their processes
```

#### User privilege control
```bash
# Create a limited user for day-to-day admin (no sudo by default)
sudo adduser secuser
sudo usermod -aG sudo daniel   # Only main admin gets sudo

# Lock the root account password (force sudo use instead)
sudo passwd -l root

# Audit sudo access
sudo cat /etc/sudoers
sudo cat /etc/group | grep sudo

# Check for users with UID 0 (should only be root)
awk -F: '$3 == 0 {print}' /etc/passwd
```

---

## ✅ Validation & Testing

Every control was verified — configuration alone isn't enough. Here's how each was tested:

### Firewall Validation (from Kali Linux)
```bash
# Port scan to confirm only allowed ports are open
nmap -sV -p- 192.168.x.20

# Expected open ports: 22 (or 2222), 80, 443
# Expected closed: everything else (3306, 8080, etc.)
```

### SSH Hardening Validation
```bash
# Confirm password auth is rejected
ssh -o PubkeyAuthentication=no daniel@192.168.x.20
# → Permission denied (publickey)

# Confirm root login is rejected
ssh root@192.168.x.20
# → Permission denied (publickey)

# Confirm key-based login succeeds
ssh -i ~/.ssh/homelab_key -p 2222 daniel@192.168.x.20
# → Successful login
```

### SSL/TLS Validation
```bash
# Check certificate details
openssl x509 -in /etc/ssl/certs/homelab.crt -text -noout | grep -E "Subject|Validity|Not"

# Test HTTPS connection
curl -vk https://homelab.local 2>&1 | grep -E "SSL|TLS|subject|issuer"

# Confirm HTTP redirects to HTTPS
curl -I http://homelab.local
# → HTTP/1.1 301 Moved Permanently
# → Location: https://homelab.local/
```

### Service Attack Surface
```bash
# Before hardening
sudo ss -tulnp | wc -l   # Count listening ports

# After hardening
sudo ss -tulnp | wc -l   # Should be significantly fewer
sudo ss -tulnp            # Manually verify each remaining service
```

---

## 📊 Hardening Results

| Control | Before | After | Verified By |
|---|---|---|---|
| Default firewall policy | Allow all | Deny all inbound | `nmap` port scan |
| Open ports | 10+ | 3 (22, 80, 443) | `nmap -p-` |
| SSH authentication | Password allowed | Key-only | Failed password attempt |
| Root SSH login | Allowed | Blocked | `ssh root@...` rejected |
| HTTP traffic | Plaintext | Forced HTTPS redirect | `curl -I http://...` |
| Encryption standard | None | TLS 1.2+ only | `openssl s_client` |
| Running services | 15+ | 8 essential | `systemctl list-units` |
| UID 0 accounts | 1 (root) | 1 (root) | `awk` /etc/passwd check |

---

## 📸 Screenshots

> *Add screenshots to `/screenshots/` directory:*
> - `01-ufw-status.png` — UFW rules output (`sudo ufw status verbose`)
> - `02-nmap-before.png` — Port scan before hardening
> - `03-nmap-after.png` — Port scan after hardening (only 3 ports open)
> - `04-ssh-password-rejected.png` — Password auth denied
> - `05-ssh-key-success.png` — Key-based login succeeding
> - `06-ssl-cert.png` — HTTPS with certificate details in browser
> - `07-http-redirect.png` — HTTP → HTTPS redirect (`curl -I`)
> - `08-services-before-after.png` — Listening ports before and after

---

## 🧠 Key Takeaways

1. **Default deny is non-negotiable** — starting with `ufw default deny incoming` and only opening what you need eliminates entire categories of risk before you configure anything else.
2. **Password SSH is a liability** — switching to key-based auth and disabling passwords removed brute-force as a viable attack vector entirely.
3. **Verification closes the loop** — it's easy to think a control is working when it isn't. Running `nmap` from an external machine and actively attempting rejected logins confirmed the hardening actually held up.
4. **Attack surface reduction compounds** — disabling unnecessary services, locking root, and restricting ports together make the system dramatically harder to exploit, even if one layer is bypassed.

---

## 🔗 Related Projects

- [SafeLine WAF Lab](https://github.com/Dcmoretz/safeline-waf-lab) — Web application attack & defense on top of this same LAMP stack
- [LAMP Stack App](https://github.com/Dcmoretz/lamp-notes-app) — The web server environment this hardening was applied to

---

## ⚠️ Disclaimer

All testing in this lab was conducted against virtual machines I own and control, in an isolated VirtualBox network. These techniques are documented for educational purposes. Never apply offensive techniques to systems you don't own or have explicit written permission to test.

---

*Built by Daniel Moretz · [dcmoretz00@gmail.com](mailto:dcmoretz00@gmail.com) · Aberdeen, NC*
