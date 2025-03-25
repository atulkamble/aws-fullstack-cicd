# aws-fullstack-cicd
Full guide (with code and steps) to set up a **ReactJS frontend** and **Node.js backend** on the **same Amazon Linux EC2 (t2.micro)** instance, and configure **CI/CD with GitHub**.

---

### 🛠️ Architecture Overview

- **Server**: Amazon Linux 2 (t2.micro)
- **Frontend**: ReactJS (served as static files via Nginx or Express)
- **Backend**: Node.js (Express)
- **CI/CD**: GitHub + Webhook + Shell Script

---

## ✅ Step-by-Step Setup

---

### **1. Launch Amazon Linux 2 EC2 Instance**

- Instance type: `t2.micro`
- AMI: Amazon Linux 2
- Security Group: Allow **ports 22 (SSH), 80 (HTTP), 3000, 5000** (for backend/frontend dev)

---

### **2. Connect to EC2 and Install Required Packages**

```bash
sudo yum update -y
sudo yum install -y git curl
curl -fsSL https://rpm.nodesource.com/setup_18.x | sudo bash -
sudo yum install -y nodejs nginx
```

> ⚠️ Node 18 is stable and compatible with most React/Express apps.

---

### **3. Clone Your Repo from GitHub**

```bash
cd /home/ec2-user
git clone https://github.com/your-username/your-fullstack-repo.git
cd your-fullstack-repo
```

📁 Structure should be something like:
```
your-fullstack-repo/
│
├── client/     ← React App
└── server/     ← Node.js Express API
```

---

### **4. Build and Deploy React Frontend**

```bash
cd client
npm install
npm run build
```

> React static files will be generated in `client/build`.

---

### **5. Setup NodeJS Backend to Serve React and API**

#### **Sample ExpressJS Backend (`server/index.js`)**

```js
const express = require("express");
const path = require("path");
const app = express();
const PORT = process.env.PORT || 5000;

// Serve static files
app.use(express.static(path.join(__dirname, "../client/build")));

// API route
app.get("/api/hello", (req, res) => {
  res.json({ message: "Hello from Node.js backend!" });
});

// Catch-all to serve React app
app.get("*", (req, res) => {
  res.sendFile(path.join(__dirname, "../client/build/index.html"));
});

app.listen(PORT, () => {
  console.log(`Server listening on port ${PORT}`);
});
```

---

### **6. Install PM2 to Keep Server Running**

```bash
sudo npm install -g pm2
cd /home/ec2-user/your-fullstack-repo/server
npm install
pm2 start index.js
pm2 startup
pm2 save
```

---

### **7. Setup Nginx (Optional - Reverse Proxy)**

Edit Nginx config:

```bash
sudo nano /etc/nginx/nginx.conf
```

Add this inside the `server` block (or default block):

```nginx
server {
    listen 80;
    server_name your-ec2-public-ip;

    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Then restart nginx:

```bash
sudo systemctl restart nginx
```

---

### **8. Setup CI/CD with GitHub**

---

#### **A. On EC2 – Create a Deployment Script**

`deploy.sh` (in your repo root):

```bash
#!/bin/bash
cd /home/ec2-user/your-fullstack-repo
git pull origin main

# Frontend
cd client
npm install
npm run build

# Backend
cd ../server
npm install
pm2 restart index.js
```

Make it executable:
```bash
chmod +x deploy.sh
```

---

#### **B. Setup GitHub Webhook**

1. Go to your GitHub repo → **Settings → Webhooks**
2. Add webhook:
   - **Payload URL**: `http://<your-ec2-ip>/webhook`
   - **Content type**: `application/json`
   - Secret: leave blank or use one

---

#### **C. Setup a Webhook Listener (mini Express app)**

Create `webhook-server.js`:

```js
const http = require('http');
const { exec } = require('child_process');

http.createServer((req, res) => {
  if (req.method === 'POST' && req.url === '/webhook') {
    let body = '';
    req.on('data', chunk => body += chunk);
    req.on('end', () => {
      exec('/home/ec2-user/your-fullstack-repo/deploy.sh', (err, stdout, stderr) => {
        if (err) console.error(`Deploy error: ${stderr}`);
        else console.log(`Deploy output:\n${stdout}`);
      });
      res.end('Webhook received and deploy started.\n');
    });
  } else {
    res.statusCode = 404;
    res.end();
  }
}).listen(3000, () => console.log('Webhook listener on port 3000'));
```

Run the webhook listener:
```bash
nohup node webhook-server.js &
```

---

### ✅ Test the Setup

1. Access your app at: `http://<EC2-IP>`
2. Make a code push to GitHub → Webhook triggers deploy script.

---

## 📦 BONUS: Save Everything to PM2

```bash
pm2 start webhook-server.js
pm2 save
```

---

