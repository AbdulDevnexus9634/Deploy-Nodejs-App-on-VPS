# Deploy-Nodejs-App-on-VPS

# 🚀 SR-HR-Backend Deployment Guide (Node.js + PM2 + GitHub Actions + Nginx)

This guide explains how to deploy your **SR-HR-Backend** Node.js project to an **Ubuntu VPS** using **GitHub Actions (CI/CD)**, **PM2**, and **Nginx** for production and staging environments.

---

## 🧩 A. Install Node.js + npm on VPS

### 1. Update your system
```bash
sudo apt update && sudo apt upgrade -y
### 2. Install Node.js (LTS 20.x recommended)
bash
Copy code
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
3. Verify installation
bash
Copy code
node -v
npm -v
✅ Expected output:

Copy code
v20.x.x
10.x.x
4. (Optional) Install PM2 globally
PM2 ensures your app keeps running even after server reboots.

bash
Copy code
sudo npm install -g pm2
pm2 -v
🧑‍💻 B. Prepare Your VPS
1. SSH into your VPS
bash
Copy code
ssh root@31.97.61.6
2. Create a non-root deploy user
bash
Copy code
adduser rozgaruser
usermod -aG sudo rozgaruser
3. Switch to the new user
bash
Copy code
su - rozgaruser
🔑 C. Setup SSH Keys for GitHub Actions
a. Generate SSH keypair locally (on your system)
Run on your local computer (Windows PowerShell):

bash
Copy code
ssh-keygen -t ed25519 -C "github-actions"
When prompted for filename:

makefile
Copy code
C:\Users\Amir\.ssh\github_actions
Leave passphrase empty.

It generates:

Private key → github_actions

Public key → github_actions.pub

b. Copy public key to VPS
From PowerShell:

bash
Copy code
scp C:\Users\Amir\.ssh\github_actions.pub rozgaruser@31.97.61.6:~
SSH into the VPS:

bash
Copy code
ssh rozgaruser@31.97.61.6
Move and set permissions:

bash
Copy code
mkdir -p ~/.ssh
chmod 700 ~/.ssh
cat ~/github_actions.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
rm ~/github_actions.pub
c. Test SSH login
From your local computer:

bash
Copy code
ssh -i C:\Users\Amir\.ssh\github_actions rozgaruser@31.97.61.6
✅ You should log in without entering a password.

d. Add private key to GitHub
Open your private key:

bash
Copy code
notepad C:\Users\Amir\.ssh\github_actions
Copy everything between:

vbnet
Copy code
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
Go to
GitHub → SR-HR-Backend → Settings → Secrets and variables → Actions → New repository secret

Add:

Name: SSH_PRIVATE_KEY

Value: (paste your private key)

📁 D. Setup Project Directories on VPS
1. Create folders for production and QA
bash
Copy code
sudo mkdir -p /var/www/prod
sudo mkdir -p /var/www/qa
sudo chown -R rozgaruser:rozgaruser /var/www
2. Clone repositories
Production
bash
Copy code
cd /var/www/prod
git clone -b main https://github.com/AbdulAzeem404/SR-HR-Backend.git .
npm install
QA (Staging)
bash
Copy code
cd /var/www/qa
git clone -b qa https://github.com/AbdulAzeem404/SR-HR-Backend.git .
npm install
⚙️ E. Manage Processes with PM2
Start applications
Production
bash
Copy code
cd /var/www/prod
pm2 start src/index.js --name prod-app
QA
bash
Copy code
cd /var/www/qa
pm2 start src/index.js --name qa-app
Save and enable on reboot
bash
Copy code
pm2 save
pm2 startup
✅ PM2 will now auto-start your apps after a reboot.

🤖 F. GitHub Actions Workflow for CI/CD Deployment
Create a file:
.github/workflows/deploy.yml

yaml
Copy code
name: Deploy to VPS

on:
  push:
    branches:
      - main
      - qa

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.7.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      # Deploy main branch (Production)
      - name: Deploy main branch to Production
        if: github.ref == 'refs/heads/main'
        run: |
          ssh -o StrictHostKeyChecking=no rozgaruser@31.97.61.6 "
          cd /var/www/prod/SR-HR-Backend &&
          git fetch origin main &&
          git reset --hard origin/main &&
          npm install &&
          pm2 restart prod-app || pm2 start src/index.js --name prod-app
          "

      # Deploy qa branch (Staging)
      - name: Deploy qa branch to Staging
        if: github.ref == 'refs/heads/qa'
        run: |
          ssh -o StrictHostKeyChecking=no rozgaruser@31.97.61.6 "
          cd /var/www/qa/SR-HR-Backend &&
          git fetch origin qa &&
          git reset --hard origin/qa &&
          npm install &&
          pm2 restart qa-app || pm2 start src/index.js --name qa-app
          "
🌐 G. Nginx Configuration
1. Create Nginx config file
Example for two subdomains:

Production (api.srhr.example.com)
nginx
Copy code
server {
    listen 80;
    server_name api.srhr.example.com;

    location / {
        proxy_pass http://localhost:3000;  # prod-app port
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
QA (qa-api.srhr.example.com)
nginx
Copy code
server {
    listen 80;
    server_name qa-api.srhr.example.com;

    location / {
        proxy_pass http://localhost:4000;  # qa-app port
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
2. Enable and reload Nginx
bash
Copy code
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
3. (Optional) Add SSL (HTTPS)
bash
Copy code
sudo certbot --nginx -d api.srhr.example.com -d qa-api.srhr.example.com
🔄 H. Workflow Summary
Branch	VPS Path	PM2 Process	Domain
main	/var/www/prod/SR-HR-Backend	prod-app	api.srhr.example.com
qa	/var/www/qa/SR-HR-Backend	qa-app	qa-api.srhr.example.com

💡 Deployment Flow
Push to qa branch → GitHub Actions deploys to /var/www/qa → accessible at qa-api.srhr.example.com

Test QA environment

Merge/push to main → deploys automatically to /var/www/prod → accessible at api.srhr.example.com

🧠 Useful PM2 Commands
Command	Description
pm2 list	Show all running apps
pm2 logs	View live logs
pm2 restart all	Restart all apps
pm2 stop all	Stop all apps
pm2 save	Save current processes
pm2 startup	Enable auto-start on reboot

✅ Deployment Summary
GitHub Actions: Automates code deployment to VPS

PM2: Keeps apps running continuously

Nginx: Serves as a reverse proxy

Certbot (optional): Adds free SSL certificates

👨‍💻 Author
Abdul Azeem
Aspiring Developer | Skilled in Node.js, Express, MongoDB, React, and Cloud Deployment

Stack Used: Node.js • Express.js • MongoDB • PM2 • Nginx • GitHub Actions
VPS IP: 31.97.61.6
Deploy User: rozgaruser

yaml
Copy code
