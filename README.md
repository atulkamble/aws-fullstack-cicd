Complete Guide to **install Node.js on EC2 and configure CI/CD with GitHub** using a simple approach — ideal for projects that don’t use AWS CodePipeline but still want auto-deployment via GitHub.

---

## ✅ **Part 1: Launch & Connect to EC2**

1. Launch EC2 (Amazon Linux 2023 or Ubuntu)
2. Allow ports **22 (SSH)**, **3000**, and/or **80** in Security Group
3. Connect via SSH:
```bash
ssh -i your-key.pem ec2-user@your-ec2-public-ip    # Amazon Linux
# or
ssh -i your-key.pem ubuntu@your-ec2-public-ip      # Ubuntu
```

---

## 📦 **Part 2: Install Node.js and Git**

### For **Amazon Linux 2023**
```bash
sudo dnf install -y nodejs npm git
```

### For **Ubuntu**
```bash
sudo apt update
sudo apt install -y nodejs npm git
```

Check:
```bash
node -v
npm -v
git --version
```

---

## 🗂️ **Part 3: Clone GitHub Repo and Setup App**

1. Clone your app repo:
```bash
git clone https://github.com/your-username/your-nodejs-app.git
cd your-nodejs-app
```

2. Install dependencies:
```bash
npm install
```

3. Test your app:
```bash
node app.js
# Or whatever your start command is
```

---

## 🔁 **Part 4: Setup PM2 to Keep App Running**

```bash
sudo npm install -g pm2
pm2 start app.js     # or npm start or whatever runs your app
pm2 save
pm2 startup
```

---

## ⚙️ **Part 5: Configure CI/CD via GitHub Webhooks**

You’ll auto-deploy the app whenever a push is made to GitHub.

### 🛠️ Step 1: Setup a Deployment Script
Create a script to pull the latest code and restart the app:

```bash
nano ~/deploy.sh
```

Paste:
```bash
#!/bin/bash
cd /home/ec2-user/your-nodejs-app
git pull origin main
npm install
pm2 restart all
```

Make it executable:
```bash
chmod +x ~/deploy.sh
```

---

### 🌐 Step 2: Install `ngrok` or Configure Nginx + SSL (Optional for HTTPS)

For GitHub Webhooks to work, you need an HTTPS endpoint. If you don’t have a domain with SSL, you can temporarily use `ngrok`:

```bash
npm install -g ngrok
ngrok http 3000
```

Use the HTTPS URL it gives for webhook in GitHub.

---

### 🔐 Step 3: Create a GitHub Webhook

1. Go to your GitHub repo → **Settings** → **Webhooks**
2. Add a webhook:
   - Payload URL: `http://your-ec2-ip:4000/webhook` (or your `ngrok` HTTPS URL)
   - Content type: `application/json`
   - Secret: *(optional)*
   - Event: `Just the push event`

---

### 🧠 Step 4: Create a Webhook Listener (using Express)

Install Express:
```bash
npm install express body-parser
```

Create `webhook.js`:
```js
const express = require('express');
const bodyParser = require('body-parser');
const { exec } = require('child_process');

const app = express();
app.use(bodyParser.json());

app.post('/webhook', (req, res) => {
    console.log('GitHub Webhook triggered');
    exec('sh ~/deploy.sh', (error, stdout, stderr) => {
        if (error) {
            console.error(`Exec error: ${error}`);
            return res.sendStatus(500);
        }
        console.log(stdout);
        res.sendStatus(200);
    });
});

app.listen(4000, () => {
    console.log('Webhook listener running on port 4000');
});
```

Run it:
```bash
node webhook.js
```

You can also run this with PM2:
```bash
pm2 start webhook.js
```

---

## 🎉 That’s It!

Now every time you push code to GitHub, it will:
- Trigger the webhook
- Pull the latest code
- Reinstall dependencies (if needed)
- Restart the app with PM2

---

Would you like this as a **full GitHub repo template**, or want to use **AWS CodePipeline** instead for a managed CI/CD setup?
