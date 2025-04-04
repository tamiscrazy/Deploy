# AWS Node.js Deployment Guide

This guide walks you through deploying a Node.js application on AWS EC2, setting up continuous operation with PM2, and configuring Caddy as a reverse proxy with HTTPS.

## Prerequisites

- An AWS EC2 instance (Ubuntu)
- SSH access to your instance (`.pem` key file)
- A domain name pointed to your EC2 instance
- Your Node.js application code in a Git repository

## Step 1: Connect to Your EC2 Instance

```bash
ssh -i "/path/to/your-key.pem" ubuntu@your-instance-ip
```

## Step 2: Update System and Install Dependencies

```bash
# Update package lists
sudo apt update

# Install Git
sudo apt install -y git

# Install Node.js 18.x
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Install PM2 for process management
sudo npm install -g pm2
```

## Step 3: Clone and Set Up Your Application

```bash
# Clone your repository
git clone https://github.com/username/your-repo.git

# Navigate to project directory
cd your-repo

# Install dependencies
npm i

# Create .env file (add your environment variables)
touch .env
# Use nano or vim to edit the .env file with your configuration
# nano .env
```

## Step 4: Configure CORS (in your application code)

Add this to your Express application:

```javascript
const cors = require('cors');
app.use(cors({ 
  origin: 'https://your-frontend-domain.com' 
}));
```

## Step 5: Start Your Application with PM2

```bash
# Start your application
pm2 start index.js

# Set PM2 to start on system boot
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu --hp /home/ubuntu

# Save the current PM2 process list
pm2 save

# View application logs
pm2 logs
```

## Step 6: Set Up Automatic Security Updates

```bash
# Install unattended-upgrades
sudo apt install -y unattended-upgrades

# Configure unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

## Step 7: Install and Configure Caddy Server

```bash
# Install required packages
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl

# Add Caddy repository
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list

# Update and install Caddy
sudo apt update
sudo apt install caddy

# Start Caddy service
sudo systemctl start caddy

# Verify Caddy is running
curl localhost
```

## Step 8: Configure Caddy as a Reverse Proxy

```bash
# Edit Caddyfile
sudo nano /etc/caddy/Caddyfile
```

Replace the contents with:

```
your-domain.com {
    reverse_proxy localhost:YOUR_NODE_APP_PORT
}
```

Save and restart Caddy:

```bash
sudo systemctl restart caddy
```

## Step 9: Set Up DNS Records

1. Create an A record pointing your domain to your EC2 IP address
2. For your frontend, create a CNAME record if needed

## Step 10: Testing

```bash
# Ping your domain to verify DNS setup
ping your-domain.com

# Restart your application if needed
pm2 restart all
```

Visit your domain in a browser. You should see your application running securely with HTTPS (Caddy handles this automatically).

## Updating Your Application

To update your application, use rsync to transfer files from your local machine:

```bash
rsync -avz \
  --exclude 'node_modules' \
  --exclude 'logs' \
  --exclude '.git' \
  --exclude '.env' \
  -e "ssh -i /path/to/your-key.pem" \
  . ubuntu@your-instance-ip:~/your-repo
```

Then SSH into your server and restart the application:

```bash
ssh -i "/path/to/your-key.pem" ubuntu@your-instance-ip
cd your-repo
npm i  # If dependencies changed
pm2 restart all
```

## Troubleshooting

- Check application logs: `pm2 logs`
- Check Caddy logs: `sudo journalctl -u caddy`
- Verify your firewall allows traffic on ports 80 and 443
