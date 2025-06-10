---
title: GitLab in Podman (Simplified)
author: Anders Karlsson
---

# GitLab in Podman - Simplified Version

## OS and Software

### OS - Debian 12 Bookworm

1. **Install SSH-key for admin user**:

   ```bash
   ssh-copy-id admin@gitlab
   ```

2. **Test SSH access**:

   ```bash
   ssh admin@gitlab
   ```

### Software Installation

1. **Update and Install Basic Software**:

   ```bash
   # Update system
   sudo apt update && sudo apt -y upgrade
   
   # Install required packages
   sudo apt -y install gpg sudo podman curl openssl
   ```

2. **Create GitLab User**:

   ```bash
   # Add a dedicated user for GitLab
   sudo useradd -m -s /bin/bash -c "GitLab User" gitlab
   
   # Enable persistent user session for GitLab
   sudo loginctl enable-linger gitlab
   ```

3. **Set up SSH keys for GitLab user**:

   ```bash
   ssh-copy-id gitlab@gitlab
   ```

## File System Setup

### Create Directory Structure

1. **Create GitLab data directories**:

   ```bash
   # Create the main GitLab directory structure
   sudo mkdir -p /opt/gitlab/{config,logs,data}
   
   # Set correct ownership
   sudo chown -R gitlab:gitlab /opt/gitlab
   ```

## GitLab Installation

1. **Log in as GitLab user**:

   ```bash
   ssh gitlab@gitlab
   ```

2. **SSL Certificates**:
   
   Create self-signed certificates (we'll replace with Let's Encrypt later):
   
   ```bash
   # Create SSL directory
   mkdir -p /opt/gitlab/config/ssl/
   
   # Create self-signed certificate
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
     -keyout /opt/gitlab/config/ssl/gitlab.mycompany.com.key \
     -out /opt/gitlab/config/ssl/gitlab.mycompany.com.crt \
     -subj "/CN=gitlab.mycompany.com"
   
   # Create certificate chain
   cp /opt/gitlab/config/ssl/gitlab.mycompany.com.crt \
      /opt/gitlab/config/ssl/gitlab.mycompany.com.chain
   ```

3. **Create GitLab Pod Configuration**:

   Choose one of the configurations below based on your authentication needs:

   **Option A: With LDAP Authentication (Enterprise/Corporate)**

   ```bash
   vi gitlab-pod-ldap.yaml
   ```

   Add the following content:

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: gitlab
     labels:
       app: gitlab
   spec:
     restartPolicy: OnFailure
     containers:
       - name: container
         image: gitlab/gitlab-ce:latest
         ports:
           - containerPort: 80
             hostPort: 8080
           - containerPort: 443
             hostPort: 8443
           - containerPort: 22
             hostPort: 2222
         volumeMounts:
           - mountPath: /etc/gitlab
             name: gitlab_config
           - mountPath: /var/log/gitlab
             name: gitlab_logs
           - mountPath: /var/opt/gitlab
             name: gitlab_data
           - mountPath: /run/secrets/ldap_password
             name: ldap-password-file
             readOnly: true
         env:
           - name: GITLAB_THEME
             value: "dark"
           - name: GITLAB_DEFAULT_THEME
             value: "dark"
           - name: GITLAB_OMNIBUS_CONFIG
             value: |
               external_url 'https://gitlab.mycompany.com:8443';
               nginx['listen_port'] = 443;
               nginx['listen_https'] = true;
               letsencrypt['enable'] = false;
               gitlab_rails['gitlab_shell_ssh_port'] = 2222;
               nginx['ssl_certificate'] = '/etc/gitlab/ssl/gitlab.mycompany.com.crt';
               nginx['ssl_certificate_key'] = '/etc/gitlab/ssl/gitlab.mycompany.com.key';
               nginx['ssl_client_certificate'] = '/etc/gitlab/ssl/gitlab.mycompany.com.chain';
               gitlab_rails['ldap_enabled'] = true;
               gitlab_rails['ldap_servers'] = {
                 'main' => {
                   'label' => 'LDAP',
                   'host' => 'dc1.mycompany.com',
                   'port' => 389,
                   'uid' => 'sAMAccountName',
                   'encryption' => 'start_tls',
                   'verify_certificates' => false,
                   'bind_dn' => 'CN=svc-ldap_reader,OU=Service Accounts,OU=Company,DC=mycompany,DC=com',
                   'password' => File.read('/run/secrets/ldap_password').strip,
                   'active_directory' => true,
                   'base' => 'DC=mycompany,DC=com',
                   'user_filter' => '',
                   'group_base' => 'OU=Groups,OU=Company,DC=mycompany,DC=com',
                   'admin_group' => 'gitlab-admins'
                 }
               };
               gitlab_rails['smtp_enable'] = true;
               gitlab_rails['smtp_address'] = "smtp.mycompany.com";
               gitlab_rails['smtp_port'] = 25;
               gitlab_rails['smtp_domain'] = "mycompany.com";
               gitlab_rails['smtp_enable_starttls_auto'] = false;
               gitlab_rails['gitlab_email_from'] = "gitlab@gitlab.mycompany.com";
               gitlab_rails['gitlab_email_reply_to'] = "noreply@gitlab.mycompany.com";
     volumes:
       - name: gitlab_config
         hostPath:
           path: /opt/gitlab/config
           type: Directory
       - name: gitlab_logs
         hostPath:
           path: /opt/gitlab/logs
           type: Directory
       - name: gitlab_data
         hostPath:
           path: /opt/gitlab/data
           type: Directory
       - name: ldap-password-file
         hostPath:
           path: /home/gitlab/ldap_pass
           type: File
   ```

   **Option B: Without LDAP (Home Lab/Personal Use)**

   ```bash
   vi gitlab-pod.yaml
   ```

   Add the following simplified configuration:

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: gitlab
     labels:
       app: gitlab
   spec:
     restartPolicy: OnFailure
     containers:
       - name: container
         image: gitlab/gitlab-ce:latest
         ports:
           - containerPort: 80
             hostPort: 8080
           - containerPort: 443
             hostPort: 8443
           - containerPort: 22
             hostPort: 2222
         volumeMounts:
           - mountPath: /etc/gitlab
             name: gitlab_config
           - mountPath: /var/log/gitlab
             name: gitlab_logs
           - mountPath: /var/opt/gitlab
             name: gitlab_data
         env:
           - name: GITLAB_THEME
             value: "dark"
           - name: GITLAB_DEFAULT_THEME
             value: "dark"
           - name: GITLAB_OMNIBUS_CONFIG
             value: |
               external_url 'https://gitlab.mycompany.com:8443';
               nginx['listen_port'] = 443;
               nginx['listen_https'] = true;
               letsencrypt['enable'] = false;
               gitlab_rails['gitlab_shell_ssh_port'] = 2222;
               nginx['ssl_certificate'] = '/etc/gitlab/ssl/gitlab.mycompany.com.crt';
               nginx['ssl_certificate_key'] = '/etc/gitlab/ssl/gitlab.mycompany.com.key';
               nginx['ssl_client_certificate'] = '/etc/gitlab/ssl/gitlab.mycompany.com.chain';
               # Disable LDAP for simple setup
               gitlab_rails['ldap_enabled'] = false;
               # Allow user registration (disable for production)
               gitlab_rails['gitlab_signup_enabled'] = true;
               # Configure email (optional - comment out if no email server)
               # gitlab_rails['smtp_enable'] = true;
               # gitlab_rails['smtp_address'] = "smtp.gmail.com";
               # gitlab_rails['smtp_port'] = 587;
               # gitlab_rails['smtp_user_name'] = "your-email@gmail.com";
               # gitlab_rails['smtp_password'] = "your-app-password";
               # gitlab_rails['smtp_domain'] = "smtp.gmail.com";
               # gitlab_rails['smtp_authentication'] = "login";
               # gitlab_rails['smtp_enable_starttls_auto'] = true;
               # gitlab_rails['gitlab_email_from'] = "gitlab@mycompany.com";
     volumes:
       - name: gitlab_config
         hostPath:
           path: /opt/gitlab/config
           type: Directory
       - name: gitlab_logs
         hostPath:
           path: /opt/gitlab/logs
           type: Directory
       - name: gitlab_data
         hostPath:
           path: /opt/gitlab/data
           type: Directory
   ```

4. **Setup for LDAP users only**:

   If you chose the LDAP configuration, create the password file:

   ```bash
   echo "your_ldap_password_here" > ldap_pass
   chmod 600 ldap_pass
   ```

   **Skip this step if using the simple configuration without LDAP.**

5. **Start GitLab**:

   ```bash
   # Use the appropriate YAML file based on your choice above
   podman play kube gitlab-pod.yaml
   # OR
   # podman play kube gitlab-pod-ldap.yaml
   ```

6. **Monitor startup**:

   ```bash
   # Follow the logs (startup takes 5-15 minutes)
   podman logs -f gitlab-container
   
   # Check ports when ready
   ss -tupln
   ```

   Expected output should show ports 8080, 8443, and 2222 listening.

7. **Test GitLab**:

   ```bash
   curl -Is https://gitlab.mycompany.com:8443 | head -n1
   ```

   Should return: `HTTP/2 302`

8. **Clean up password file (LDAP users only)**:

   ```bash
   # Only needed if you used the LDAP configuration
   shred -u ldap_pass
   ```

9. **Create systemd service**:

   ```bash
   podman generate systemd --new --files --name gitlab
   systemctl --user enable container-gitlab.service
   ```

## Initial Setup

### First Login

**For installations without LDAP:**

1. **Get the initial root password**:

   ```bash
   # Wait for GitLab to fully start, then get the initial password
   sudo cat /opt/gitlab/config/initial_root_password
   ```

2. **Login to GitLab**:
   - Navigate to `https://gitlab.mycompany.com`
   - Username: `root`
   - Password: (from the file above)
   - **Important**: Change the root password immediately after first login!

3. **Create your admin user**:
   - Go to Admin Area → Users → New User
   - Create your personal admin account
   - Sign out and sign in with your new account
   - Optionally disable the root account

**For LDAP installations:**
- Users can log in directly with their domain credentials
- Make sure your user is member of the `gitlab-admins` group for admin access

## Let's Encrypt Certificates

### Install Certbot

1. **Install Certbot**:

   ```bash
   sudo apt -y install certbot python3-certbot-nginx
   ```

2. **Request certificate**:

   ```bash
   sudo certbot certonly --standalone \
     --preferred-challenges http \
     -d gitlab.mycompany.com
   ```

3. **Create certificate update script**:

   ```bash
   sudo mkdir -p /root/bin
   sudo vi /root/bin/gitlab-cert-renew.sh
   ```

   Add the following content:

   ```bash
   #!/bin/bash
   
   # Copy Let's Encrypt certificates to GitLab
   cp /etc/letsencrypt/live/gitlab.mycompany.com/fullchain.pem \
      /opt/gitlab/config/ssl/gitlab.mycompany.com.crt
   cp /etc/letsencrypt/live/gitlab.mycompany.com/privkey.pem \
      /opt/gitlab/config/ssl/gitlab.mycompany.com.key
   cp /etc/letsencrypt/live/gitlab.mycompany.com/chain.pem \
      /opt/gitlab/config/ssl/gitlab.mycompany.com.chain
   
   # Set correct permissions
   chown gitlab:gitlab /opt/gitlab/config/ssl/*
   chmod 644 /opt/gitlab/config/ssl/*.crt
   chmod 644 /opt/gitlab/config/ssl/*.chain
   chmod 600 /opt/gitlab/config/ssl/*.key
   
   # Reload GitLab nginx
   runuser -l gitlab -c 'podman exec gitlab-container gitlab-ctl hup nginx'
   ```

4. **Make script executable and add to cron**:

   ```bash
   sudo chmod +x /root/bin/gitlab-cert-renew.sh
   
   # Add to crontab
   sudo crontab -e
   ```

   Add these lines:

   ```
   0 2 * * * certbot renew --quiet
   5 2 * * * /root/bin/gitlab-cert-renew.sh
   ```

## Nginx Frontend

Set up Nginx as a reverse proxy to access GitLab on standard ports (80/443).

1. **Install Nginx**:

   ```bash
   sudo apt -y install nginx
   ```

2. **Configure Nginx**:

   ```bash
   sudo vi /etc/nginx/sites-available/gitlab.mycompany.com
   ```

   Add the following:

   ```nginx
   # HTTP to HTTPS redirect
   server {
       listen 80;
       listen [::]:80;
       server_name gitlab.mycompany.com;
       return 301 https://$server_name$request_uri;
   }
   
   # HTTPS proxy to GitLab
   server {
       listen 443 ssl http2;
       listen [::]:443 ssl http2;
       server_name gitlab.mycompany.com;
   
       # SSL certificates
       ssl_certificate /etc/letsencrypt/live/gitlab.mycompany.com/fullchain.pem;
       ssl_certificate_key /etc/letsencrypt/live/gitlab.mycompany.com/privkey.pem;
   
       # Modern SSL configuration
       ssl_protocols TLSv1.2 TLSv1.3;
       ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384;
   
       # Proxy to GitLab container
       location / {
           proxy_pass https://127.0.0.1:8443;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
           proxy_set_header X-Forwarded-Ssl on;
   
           # WebSocket support
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection "upgrade";
       }
   }
   ```

3. **Enable site and restart Nginx**:

   ```bash
   sudo ln -s /etc/nginx/sites-available/gitlab.mycompany.com /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl restart nginx
   ```

## Security

### Firewall with nftables

1. **Install and configure nftables**:

   ```bash
   sudo apt -y install nftables
   sudo vi /etc/nftables.conf
   ```

   Add the following ruleset:

   ```
   #!/usr/sbin/nft -f
   
   flush ruleset
   
   table inet filter {
       chain input {
           type filter hook input priority 0; policy drop;
           
           # Allow established connections
           ct state established,related accept
   
           # Allow loopback
           iif lo accept
   
           # Allow SSH from trusted networks
           tcp dport 22 ip saddr { 192.168.1.0/24, 10.0.0.0/8 } accept
   
           # Allow HTTP/HTTPS
           tcp dport { 80, 443 } accept
   
           # Allow GitLab container ports from localhost only
           tcp dport { 8080, 8443, 2222 } ip saddr 127.0.0.1 accept
   
           # Log and drop everything else
           log prefix "nft_drop: " drop
       }
   
       chain forward {
           type filter hook forward priority 0; policy drop;
           # Allow container traffic
           ct state established,related accept
           iifname "podman*" accept
           oifname "podman*" accept
       }
   
       chain output {
           type filter hook output priority 0; policy accept;
       }
   }
   ```

2. **Apply firewall rules**:

   ```bash
   sudo nft -c -f /etc/nftables.conf  # Test configuration
   sudo nft -f /etc/nftables.conf     # Apply rules
   sudo systemctl enable nftables
   sudo systemctl restart nftables
   ```

### SSH Hardening

1. **Configure SSH**:

   ```bash
   sudo vi /etc/ssh/sshd_config
   ```

   Ensure these settings:

   ```
   Port 22
   PermitRootLogin no
   PasswordAuthentication no
   AllowUsers admin gitlab
   MaxAuthTries 3
   ```

2. **Restart SSH**:

   ```bash
   sudo systemctl restart ssh
   ```

### Automatic Security Updates

1. **Enable unattended upgrades**:

   ```bash
   sudo apt -y install unattended-upgrades
   sudo dpkg-reconfigure -plow unattended-upgrades
   ```

## Backup Strategy

### Local Backup with rsync

Create a simple backup solution using rsync:

1. **Create backup script**:

   ```bash
   sudo vi /root/bin/gitlab-backup.sh
   ```

   ```bash
   #!/bin/bash
   
   BACKUP_DIR="/backup/gitlab"
   DATE=$(date +%Y%m%d_%H%M%S)
   
   # Create backup directory
   mkdir -p "$BACKUP_DIR"
   
   # Stop GitLab (optional, for consistent backup)
   # runuser -l gitlab -c 'podman stop gitlab-container'
   
   # Backup GitLab data
   rsync -av --delete /opt/gitlab/ "$BACKUP_DIR/gitlab_$DATE/"
   
   # Keep only last 7 backups
   cd "$BACKUP_DIR"
   ls -t | tail -n +8 | xargs rm -rf
   
   # Start GitLab if stopped
   # runuser -l gitlab -c 'podman start gitlab-container'
   
   echo "Backup completed: $BACKUP_DIR/gitlab_$DATE"
   ```

2. **Make executable and schedule**:

   ```bash
   sudo chmod +x /root/bin/gitlab-backup.sh
   
   # Add to crontab for daily backups
   sudo crontab -e
   ```

   Add:

   ```
   0 3 * * * /root/bin/gitlab-backup.sh
   ```

## Troubleshooting

### Common Issues

**GitLab container won't start:**
```bash
podman logs gitlab-container
journalctl --user -u container-gitlab.service
```

**SSL certificate issues:**
```bash
openssl x509 -in /opt/gitlab/config/ssl/gitlab.mycompany.com.crt -text -noout
```

**Permission problems:**
```bash
sudo chown -R gitlab:gitlab /opt/gitlab
```

**Firewall blocking connections:**
```bash
sudo nft list ruleset
sudo journalctl | grep nft_drop
```

### Useful Commands

```bash
# Check GitLab status
podman ps -a

# Restart GitLab
systemctl --user restart container-gitlab.service

# Check resource usage
podman stats

# GitLab health check
curl -k https://localhost:8443/-/health
```

## Notes

- Replace `gitlab.mycompany.com` with your actual domain
- Update IP ranges in firewall rules to match your network
- Adjust LDAP configuration for your Active Directory setup
- Consider using external databases for production environments
- Regular backups are essential - test restore procedures periodically
