# 🚀 VPS Full Deployment Guide (Ubuntu)

Ye document **fresh Ubuntu VPS** par **complete production deployment flow** explain karta hai.

---

## 📌 Flow Overview

1. System Installation  
2. GitHub SSH Setup (Multiple Accounts)  
3. Project Setup in `/var/www/project`  
4. Backend & Frontend Run (PM2)  
5. Nginx Setup (Frontend + `/api` Backend)  
6. Firewall & SSL  

---

# 🔹 1. System Installation (Base Setup)

## 1.1 Login to VPS
```bash
ssh root@YOUR_VPS_IP
```

## 1.2 Update System
sudo apt update && sudo apt upgrade -y

## 1.3 Install Basic Packages
sudo apt install -y \
curl \
wget \
git \
unzip \
build-essential \
software-properties-common


Verify Git:

git --version

##  1.4 Install Node.js (LTS)
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs


Verify:

node -v
npm -v

##  1.5 Install PM2 (Process Manager)
sudo npm install -g pm2
pm2 startup

## 1.6 Install Python
sudo apt install -y python3 python3-pip


Verify:

python3 --version
pip3 --version

## 1.7 Install MongoDB
curl -fsSL https://pgp.mongodb.com/server-6.0.asc | \
sudo gpg --dearmor -o /usr/share/keyrings/mongodb-server-6.0.gpg

echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg ] \
https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse" | \
sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list

sudo apt update
sudo apt install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod

## 1.8 Install Nginx
sudo apt install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx


Test:

http://YOUR_VPS_IP

# 2. GitHub SSH Setup (Multiple Accounts)
Introduction

Multiple GitHub accounts (personal / work / client) ke liye SSH config best approach hai.

Step 1: Generate SSH Keys
ls -al ~/.ssh


Generate personal key:

ssh-keygen -t rsa -C "personal@email.com"


Generate work key:

ssh-keygen -t rsa -C "work@email.com" -f ~/.ssh/id_rsa_work

Step 2: Add SSH Key to GitHub
cat ~/.ssh/id_rsa.pub
cat ~/.ssh/id_rsa_work.pub


GitHub →
Settings → SSH & GPG Keys → New SSH Key → Paste → Save

Step 3: SSH Config File
cd ~/.ssh
nano config

# Personal GitHub
Host github.com-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_rsa

# Work GitHub
Host github.com-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_rsa_work

Step 4: Add Keys to SSH Agent
ssh-add -D
ssh-add ~/.ssh/id_rsa
ssh-add ~/.ssh/id_rsa_work


Test:

ssh -T git@github.com-personal
ssh -T git@github.com-work

# 3. Project Setup in VPS
cd /var/www
sudo mkdir project
sudo chown -R $USER:$USER project
cd project


Clone repository:

git clone git@github.com-work:username/repo.git .


Install dependencies:

npm install
npm run build

# 4. Run Backend & Frontend Using PM2
Backend (Node / Express)
pm2 start index.js --name backend


Backend runs on:

http://localhost:5000

Frontend (Next.js / React)
pm2 start npm --name frontend -- start


Frontend runs on:

http://localhost:3000


Save PM2:

pm2 save

# 5. Nginx Setup (Frontend + /api Backend)
🔍 Port Mapping Explanation
Browser → Nginx (80 / 443)
          ├── /        → Frontend (3000)
          └── /api     → Backend  (5000)

Create Nginx Config
sudo nano /etc/nginx/sites-available/project

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    # Frontend
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # Backend API
    location /api/ {
        proxy_pass http://localhost:5000/api/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

Enable config:

sudo ln -s /etc/nginx/sites-available/project /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx

# 6. Firewall Setup
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable

# 7. SSL Setup (HTTPS)
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com


Verify auto-renew:

sudo certbot renew --dry-run

# 8. Useful Commands
pm2 status
pm2 logs
sudo tail -f /var/log/nginx/error.log
sudo journalctl -u mongod