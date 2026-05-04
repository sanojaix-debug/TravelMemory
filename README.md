# TravelMemory — MERN Stack Deployment on AWS

**HeroVired DevOps Assignment**  
**Deployed by:** Sanoj Ithanasekaran  
**Date:** May 2026

---

## 🏗️ Architecture Overview

```
User (Browser)
      ↓
AWS Application Load Balancer
(travelmemory-alb-626759165.ap-south-1.elb.amazonaws.com)
      ↓
┌─────────────────────┬─────────────────────┐
│  EC2 Server-1       │  EC2 Server-2       │
│  IP: 65.0.105.54    │  IP: 3.110.208.28   │
│  Nginx + Node.js    │  Nginx + Node.js    │
│  PM2 Process Mgr    │  PM2 Process Mgr    │
│  ap-south-1b        │  ap-south-1b        │
└─────────────────────┴─────────────────────┘
              ↓
       MongoDB Atlas
       SanojCluster
   AWS Mumbai (ap-south-1)
```

---

## 🔧 Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React.js |
| Backend | Node.js + Express |
| Database | MongoDB Atlas (M0 Free Tier) |
| Web Server | Nginx 1.28 (Reverse Proxy) |
| Process Manager | PM2 |
| Cloud | AWS EC2 (t2.micro) |
| Load Balancer | AWS Application Load Balancer |
| OS | Ubuntu 26.04 LTS |

---

## 📋 Prerequisites

- AWS Account with EC2 access
- MongoDB Atlas account
- Node.js 18+ installed locally
- Git installed
- SSH client (Terminal on Mac/Linux)

---

## 🚀 Deployment Steps

### Phase 1 — Local Setup & Testing

**1. Fork and clone the repository:**
```bash
git clone https://github.com/sanojaix-debug/TravelMemory
cd TravelMemory
```

**2. Backend setup:**
```bash
cd backend
npm install
nano .env
```

Add to `.env`:
```
MONGO_URI=mongodb+srv://<username>:<password>@sanojcluster.wxglth8.mongodb.net/travelmemory?appName=SanojCluster
PORT=3001
```

**3. Start backend:**
```bash
node index.js
```

**4. Frontend setup:**
```bash
cd ../frontend
npm install
nano .env
```

Add to `.env`:
```
REACT_APP_BACKEND_URL=http://localhost:3001
```

**5. Start frontend:**
```bash
npm start
```

---

### Phase 2 — AWS EC2 Instance Setup

**Instance details:**
- AMI: Ubuntu 26.04 LTS
- Instance type: t2.micro
- Region: ap-south-1 (Mumbai)
- Security Group ports: 22 (SSH), 80 (HTTP), 3001 (Node.js)

**Install dependencies on EC2:**
```bash
sudo apt update -y
sudo apt install -y nodejs npm git nginx
sudo npm install -g pm2
```

**Clone repo and setup backend:**
```bash
git clone https://github.com/sanojaix-debug/TravelMemory
cd TravelMemory/backend
npm install
nano .env  # Add MONGO_URI and PORT
```

**Start with PM2:**
```bash
pm2 start index.js --name travelmemory-backend
pm2 startup
pm2 save
```

---

### Phase 3 — Nginx Configuration

**Create Nginx config:**
```bash
sudo nano /etc/nginx/sites-available/travelmemory
```

```nginx
server {
    listen 80;
    server_name _;

    root /home/ubuntu/TravelMemory/frontend/build;
    index index.html;

    location / {
        try_files $uri /index.html;
    }

    location /api/ {
        proxy_pass http://localhost:3001/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

**Enable and restart:**
```bash
sudo ln -s /etc/nginx/sites-available/travelmemory /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

**Build frontend:**
```bash
cd ~/TravelMemory/frontend
npm run build
sudo chmod 755 /home/ubuntu
sudo chmod -R 755 /home/ubuntu/TravelMemory/frontend/build
```

---

### Phase 4 — Second Instance & Load Balancer

**Create AMI from Server-1:**
- EC2 Console → Select TravelMemory-Server-1
- Actions → Image and Templates → Create Image
- Name: `TravelMemory-AMI`

**Launch Server-2 from AMI:**
- EC2 → AMIs → Select TravelMemory-AMI
- Launch instance → Name: `TravelMemory-Server-2`
- Same key pair and security group

**Create Target Group:**
- Name: `travelmemory-tg`
- Protocol: HTTP | Port: 80
- Health check path: `/`
- Register both EC2 instances

**Create Application Load Balancer:**
- Name: `travelmemory-alb`
- Scheme: Internet-facing
- Listener: HTTP:80 → travelmemory-tg

**Update url.js on both instances:**
```javascript
export const baseUrl = "http://travelmemory-alb-626759165.ap-south-1.elb.amazonaws.com/api";
```

Rebuild frontend on both servers:
```bash
cd ~/TravelMemory/frontend
npm run build
sudo systemctl restart nginx
```

---

## 🌐 Live URLs

| Resource | URL |
|----------|-----|
| Server-1 (Direct) | http://65.0.105.54 |
| Server-2 (Direct) | http://3.110.208.28 |
| Load Balancer | http://travelmemory-alb-626759165.ap-south-1.elb.amazonaws.com |

---

## 📁 Repository Structure

```
TravelMemory/
├── backend/
│   ├── index.js
│   ├── .env          (not committed)
│   └── package.json
├── frontend/
│   ├── src/
│   │   ├── url.js    (backend URL config)
│   │   └── components/
│   ├── build/        (generated)
│   └── package.json
└── README.md
```

---

## 🔍 Verification Checklist

- [x] Local app runs on localhost:3000
- [x] Backend connects to MongoDB Atlas
- [x] EC2 Server-1 running with Nginx + PM2
- [x] EC2 Server-2 created from AMI
- [x] Target Group shows both instances Healthy
- [x] ALB routes traffic to both instances
- [x] App accessible via ALB DNS URL
- [x] Add Experience form saves to MongoDB
- [ ] Cloudflare domain setup (skipped — no domain)

---

## ⚡ Quick Reference Commands

```bash
# Check backend status
pm2 status

# View backend logs
pm2 logs travelmemory-backend

# Restart backend
pm2 restart travelmemory-backend

# Rebuild frontend
cd ~/TravelMemory/frontend && npm run build

# Restart Nginx
sudo systemctl restart nginx

# Check Nginx status
sudo systemctl status nginx

# View Nginx error logs
sudo tail -20 /var/log/nginx/error.log
```

---

## 🛠️ Troubleshooting

| Issue | Fix |
|-------|-----|
| 500 Internal Server Error | `sudo chmod 755 /home/ubuntu && sudo chmod -R 755 /home/ubuntu/TravelMemory/frontend/build` |
| 502 Bad Gateway | PM2 backend not running — `pm2 restart travelmemory-backend` |
| App shows "Loading..." | Check url.js has correct backend URL, rebuild frontend |
| MongoDB connection failed | Check .env MONGO_URI, verify Network Access in Atlas |
| Target Group Unhealthy | Check Security Group has port 80 open, Nginx running |
