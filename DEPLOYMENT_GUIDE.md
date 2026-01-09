# 🚀 Deployment Guide - Node.js CI/CD to EC2

## Overview

This project uses GitHub Actions to automatically deploy a Node.js/TypeScript application to EC2 with zero-downtime deployments using PM2.

## 📋 Prerequisites

### 1. GitHub Secrets Required

Set these in your repository: `Settings` → `Secrets and variables` → `Actions`

- `EC2_HOST` - Your EC2 public IP or domain (e.g., `54.123.45.67` or `yourdomain.com`)
- `EC2_USERNAME` - SSH username (usually `ubuntu` or `ec2-user`)
- `EC2_SSH_KEY` - Your private SSH key (`.pem` file contents)

### 2. EC2 Security Group

Make sure your EC2 instance has these inbound rules:

- **Port 22** (SSH) - For deployment access
- **Port 80** (HTTP) - For nginx
- **Port 8000** (Custom TCP) - For your Node.js app (if accessing directly)

### 3. EC2 Server Setup

Your EC2 instance should have:

```bash
# Install Node.js (v20.x)
sudo apt-get update
sudo apt-get install npm -y
sudo npm i -g n
sudo n lts

curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install PM2 globally
sudo npm install -g pm2

# Setup PM2 to start on system boot
pm2 startup systemd
sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u $USER --hp /home/$USER
pm2 startup
sudo pm2 start ecosystem.config.cjs

# Create app directory
sudo mkdir -p /var/www/express-app
sudo chown -R $USER:$USER /var/www/express-app

# Optional: Setup nginx reverse proxy
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

#### Nginx Configuration (Optional)

If using nginx as reverse proxy, create - sudo nano /etc/nginx/sites-available/express-app:

```nginx
server {
    listen 80;
    server_name 16.16.90.101;  # or your EC2 IP

    location / {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/express-app /etc/nginx/sites-enabled/
## remove defailt nginx
sudo ln -s /etc/nginx/sites-available/default
## test nginx
sudo nginx -t
sudo systemctl restart nginx
```

## 🔄 Deployment Workflow

### Automatic Deployment

Push to `main` branch:

```bash
git add .
git commit -m "Your changes"
git push origin main
```

### Manual Deployment

1. Go to GitHub Actions tab
2. Select "Deploy to EC2" workflow
3. Click "Run workflow"
4. Select branch and click "Run workflow"

## 🛠️ How It Works

### Build Job

1. Checks out code
2. Sets up Node.js 20.13.1
3. Installs dependencies (including dev dependencies for TypeScript compilation)
4. Builds TypeScript → JavaScript
5. Creates deployment package with:
   - `dist/` (compiled JavaScript)
   - `package.json` & `package-lock.json`
   - `ecosystem.config.cjs` (PM2 config)
6. Uploads artifact for deployment

### Deploy Job

1. Downloads build artifact
2. Copies to EC2 via SCP
3. SSHs into EC2 and:
   - Creates backup of current version
   - Extracts new deployment package
   - Installs production dependencies (`npm ci --only=production`)
   - Creates `.env` file if needed (first deploy only)
   - **Zero-downtime reload** with PM2 (`pm2 reload`)
   - Shows PM2 status and logs
4. Verifies deployment by checking HTTP response via nginx
5. Rolls back if deployment fails

## 🎯 Zero-Downtime Deployment

The workflow uses `pm2 reload` instead of `pm2 restart`:

- **`pm2 reload`** - Gracefully reloads app in cluster mode with zero downtime
- **`pm2 restart`** - Stops and starts app (causes brief downtime)

### PM2 Cluster Mode

Your app runs in cluster mode with multiple instances:

```javascript
instances: "max",      // Uses all available CPU cores
exec_mode: "cluster"   // Enables zero-downtime reloads
```

## 🔍 Debugging

### Check PM2 Status on EC2

```bash
ssh -i your-key.pem ubuntu@your-ec2-ip

# Check PM2 processes
pm2 list

# View logs
pm2 logs express-app

# View last 100 lines
pm2 logs express-app --lines 100

# Monitor in real-time
pm2 monit
```

### Use Debug Workflow

Run the "Debug EC2 Server" workflow to get detailed diagnostics:

- System info
- Node/NPM/PM2 versions
- PM2 process status
- App logs
- Port status
- File structure

### Common Issues

#### 1. Code Not Updating

**Symptoms:** Workflow succeeds but changes don't appear
**Solution:**

- Check PM2 logs: `pm2 logs express-app`
- Verify dist folder: `ls -la /var/www/express-app/dist/`
- Check build output in GitHub Actions logs

#### 2. Port Issues

**Symptoms:** `Got response: 000`
**Solution:**

- Verify app port matches `.env` file on server
- Check EC2 Security Group has correct ports open
- Verify nginx is proxying correctly

#### 3. PM2 Not Starting

**Symptoms:** PM2 shows "errored" status
**Solution:**

```bash
cd /var/www/express-app
pm2 delete express-app
pm2 start ecosystem.config.cjs
pm2 logs express-app
```

## 📁 Project Structure

```
/var/www/express-app/
├── dist/              # Compiled JavaScript
│   └── index.js
├── node_modules/      # Production dependencies
├── logs/              # PM2 logs
│   ├── err.log
│   ├── out.log
│   └── combined.log
├── backups/           # Automatic backups (last 5)
├── package.json
├── package-lock.json
├── ecosystem.config.cjs
└── .env               # Environment variables
```

## 🔐 Environment Variables

Located at `/var/www/express-app/.env`:

```env
NODE_ENV=production
PORT=8000
```

## 📊 Monitoring

### PM2 Commands

```bash
# List all processes
pm2 list

# View logs
pm2 logs express-app

# Restart app
pm2 restart express-app

# Reload app (zero-downtime)
pm2 reload express-app

# Stop app
pm2 stop express-app

# Delete app from PM2
pm2 delete express-app

# Save PM2 process list
pm2 save

# Monitor resources
pm2 monit
```

## 🔄 Rollback

If deployment fails, the workflow automatically attempts rollback to the previous version.

Manual rollback:

```bash
cd /var/www/express-app/backups
ls -lt  # Find backup file
tar -xzf backup_YYYYMMDD_HHMMSS.tar.gz -C /var/www/express-app
cd /var/www/express-app
npm ci --only=production
pm2 reload ecosystem.config.cjs
```

## 🚨 Emergency Procedures

### Stop Application

```bash
pm2 stop express-app
```

### View Error Logs

```bash
pm2 logs express-app --err --lines 100
```

### Restart Application

```bash
pm2 restart express-app
```

### Full Reset

```bash
cd /var/www/express-app
pm2 delete all
pm2 start ecosystem.config.cjs
pm2 save
```

## 📝 Workflow Files

### Main Deployment: `.github/workflows/aws.yml`

- Triggered on push to `main` branch
- Manual trigger available
- Zero-downtime deployment with PM2 reload
- Automatic rollback on failure

### Debug Workflow: `.github/workflows/debug-ec2.yml`

- Manual trigger only
- Comprehensive diagnostics
- Safe to run anytime

## ✅ Best Practices

1. **Always test locally** before pushing to main
2. **Review GitHub Actions logs** after each deployment
3. **Keep backups** - workflow keeps last 5 automatically
4. **Monitor PM2 logs** regularly
5. **Use manual deployment** for critical releases
6. **Test rollback procedure** in staging first

## 🔗 Useful Links

- [PM2 Documentation](https://pm2.keymetrics.io/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Nginx Documentation](https://nginx.org/en/docs/)
