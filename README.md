# 🚀 SR-HR-Backend — VPS Deployment Guide
Deploy a Node.js app to an Ubuntu VPS using **PM2**, **Nginx**, and **GitHub Actions (CI/CD)** with separate **Production** (`main`) and **Staging** (`qa`) environments.

---

## 📋 Prerequisites
- Ubuntu VPS (IP: `31.97.61.6`)
- Non-root user: `rozgaruser` (sudo enabled)
- Repo: `AbdulAzeem404/SR-HR-Backend`
- Node.js LTS (20.x), npm, PM2, Nginx installed
- Domain(s) (optional): `api.srhr.example.com`, `qa-api.srhr.example.com`

---

## A) Install Node.js + npm (on VPS)
```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
node -v
npm -v
# Optional: PM2 (process manager)
sudo npm install -g pm2
pm2 -v

B) Create Deploy User (once)
ssh root@31.97.61.6
adduser rozgaruser
usermod -aG sudo rozgaruser
su - rozgaruser

C) Configure SSH for GitHub Actions
1) Generate keys on your local machine (Windows PowerShell)
ssh-keygen -t ed25519 -C "github-actions"
# File path: C:\Users\Amir\.ssh\github_actions
# Leave passphrase empty

2) Copy the public key to the VPS
scp C:\Users\Amir\.ssh\github_actions.pub rozgaruser@31.97.61.6:~
ssh rozgaruser@31.97.61.6
mkdir -p ~/.ssh && chmod 700 ~/.ssh
cat ~/github_actions.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
rm ~/github_actions.pub

3) Test key-based login (from local)
ssh -i C:\Users\Amir\.ssh\github_actions rozgaruser@31.97.61.6

4) Add the private key to GitHub Actions Secrets

Open: notepad C:\Users\Amir\.ssh\github_actions

Copy entire content (BEGIN/END OPENSSH PRIVATE KEY)

GitHub → Repo → Settings → Secrets and variables → Actions → New repository secret

Name: SSH_PRIVATE_KEY

Value: (paste private key)

D) Setup Project Directories (on VPS)
sudo mkdir -p /var/www/prod /var/www/qa
sudo chown -R rozgaruser:rozgaruser /var/www

Clone repositories
# Production
cd /var/www/prod
git clone -b main https://github.com/AbdulAzeem404/SR-HR-Backend.git .
npm install

# Staging (QA)
cd /var/www/qa
git clone -b qa https://github.com/AbdulAzeem404/SR-HR-Backend.git .
npm install

E) Run with PM2
# Production
cd /var/www/prod
pm2 start src/index.js --name prod-app

# Staging (QA)
cd /var/www/qa
pm2 start src/index.js --name qa-app

# Persist across reboots
pm2 save
pm2 startup


Common PM2 commands

pm2 list
pm2 logs
pm2 restart prod-app
pm2 restart qa-app

F) CI/CD with GitHub Actions

Create file: .github/workflows/deploy.yml

name: Deploy to VPS

on:
  push:
    branches: [ main, qa ]

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

G) Nginx Reverse Proxy (optional but recommended)

Create a site config (e.g., /etc/nginx/sites-available/myapp) and add:

Production (api.srhr.example.com)

server {
    listen 80;
    server_name api.srhr.example.com;

    location / {
        proxy_pass http://localhost:3000;  # change to your prod app port
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}


Staging (qa-api.srhr.example.com)

server {
    listen 80;
    server_name qa-api.srhr.example.com;

    location / {
        proxy_pass http://localhost:4000;  # change to your QA app port
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}


Enable and reload:

sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

HTTPS (Let’s Encrypt)
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d api.srhr.example.com -d qa-api.srhr.example.com

H) Deploy Flow
Branch	VPS Path	PM2 Process	Domain (example)
main	/var/www/prod/SR-HR-Backend	prod-app	api.srhr.example.com
qa	/var/www/qa/SR-HR-Backend	qa-app	qa-api.srhr.example.com

How it works

Push to qa → auto-deploy to /var/www/qa → test on qa-api.srhr.example.com

Merge/push to main → auto-deploy to /var/www/prod → live on api.srhr.example.com

🔧 Troubleshooting

Permission denied (publickey): Re-check authorized_keys and the GitHub secret key.

PM2 app not restarting: pm2 logs <name> to inspect errors.

Nginx 502/Bad Gateway: Confirm the app is listening on the port you proxy to.

Different start file: Update pm2 start src/index.js to your entry point.

🧑‍💻 Author

Abdul Azeem
Stack: Node.js • Express • MongoDB • PM2 • Nginx • GitHub Actions
VPS: 31.97.61.6 • User: rozgaruser
