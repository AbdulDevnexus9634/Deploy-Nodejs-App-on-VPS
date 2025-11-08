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

