# Python TTS Service - VPS Deployment Guide (اردو)

## 📋 مکمل Deployment گائیڈ

### Step 1: VPS پر ابتدائی Setup

```bash
# VPS میں SSH کریں
ssh root@your-vps-ip

# System update کریں
sudo apt update && sudo apt upgrade -y

# Python اور dependencies install کریں
sudo apt install python3 python3-pip python3-venv nginx supervisor -y

# Git install کریں (اگر نہیں ہے)
sudo apt install git -y
```

### Step 2: Project Setup

```bash
# Project directory میں جائیں
cd /var/www/

# اگر آپ نے پہلے سے clone نہیں کیا
git clone https://github.com/TechSolutionor98/ai_voice_caller.git
cd ai_voice_caller/python-tts-service

# یا اگر پہلے سے clone کیا ہوا ہے
cd python-tts-service
```

### Step 3: Virtual Environment Setup

```bash
# Virtual environment بنائیں
python3 -m venv venv

# Activate کریں
source venv/bin/activate

# Dependencies install کریں
pip install --upgrade pip
pip install -r requirements.txt

# Additional system dependencies
sudo apt install espeak espeak-data libespeak-dev -y
sudo apt install ffmpeg -y
```

### Step 4: Environment Configuration

```bash
# .env file بنائیں
nano .env
```

**.env file میں یہ add کریں:**
```env
PORT=5001
FLASK_ENV=production
HOST=0.0.0.0
```

### Step 5: Test کریں کہ Service چل رہی ہے

```bash
# Virtual environment activate کریں
source venv/bin/activate

# App چلائیں
python app.py

# دوسرے terminal میں test کریں
curl http://localhost:5001/health
```

اگر `{"status": "healthy"}` آئے تو service کام کر رہی ہے! `Ctrl+C` سے بند کریں۔

### Step 6: Supervisor Setup (Service Auto-Start)

```bash
# Supervisor config بنائیں
sudo nano /etc/supervisor/conf.d/python-tts.conf
```

**یہ configuration add کریں:**
```ini
[program:python-tts]
directory=/var/www/ai_voice_caller/python-tts-service
command=/var/www/ai_voice_caller/python-tts-service/venv/bin/python app.py
user=root
autostart=true
autorestart=true
stderr_logfile=/var/log/python-tts/err.log
stdout_logfile=/var/log/python-tts/out.log
environment=PATH="/var/www/ai_voice_caller/python-tts-service/venv/bin"
```

```bash
# Log directory بنائیں
sudo mkdir -p /var/log/python-tts

# Supervisor کو reload کریں
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start python-tts

# Status چیک کریں
sudo supervisorctl status python-tts
```

### Step 7: Nginx Reverse Proxy Setup

```bash
# Nginx config بنائیں
sudo nano /etc/nginx/sites-available/tts
```

**Configuration (بغیر Domain کے - صرف IP سے):**
```nginx
server {
    listen 80;
    server_name your-vps-ip;

    location /tts/ {
        proxy_pass http://localhost:5001/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # CORS headers
        add_header 'Access-Control-Allow-Origin' '*' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Origin, Content-Type, Accept, Authorization' always;
        
        # Handle OPTIONS requests
        if ($request_method = 'OPTIONS') {
            return 204;
        }
    }
}
```

**Configuration (Domain کے ساتھ):**
```nginx
server {
    listen 80;
    server_name tts.yourdomain.com;

    location / {
        proxy_pass http://localhost:5001/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # CORS headers
        add_header 'Access-Control-Allow-Origin' '*' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Origin, Content-Type, Accept, Authorization' always;
        
        # Handle OPTIONS requests
        if ($request_method = 'OPTIONS') {
            return 204;
        }
    }
}
```

```bash
# Config enable کریں
sudo ln -s /etc/nginx/sites-available/tts /etc/nginx/sites-enabled/

# Nginx test کریں
sudo nginx -t

# Nginx restart کریں
sudo systemctl restart nginx
```

### Step 8: Domain Setup (اختیاری لیکن تجویز کردہ)

#### Domain DNS Settings:

1. **اپنے Domain Provider پر جائیں** (Namecheap, GoDaddy, Cloudflare وغیرہ)
2. **DNS Records add کریں:**

```
Type: A Record
Name: tts (یا @ اگر main domain استعمال کرنا ہے)
Value: your-vps-ip
TTL: 3600 یا Auto
```

**مثال:**
- اگر آپ کا domain `yourdomain.com` ہے
- A Record: `tts` → `123.45.67.89` (آپ کی VPS IP)
- Result: `tts.yourdomain.com` آپ کی TTS service پر point کرے گا

#### DNS Propagation چیک کریں:
```bash
# 5-30 منٹ انتظار کریں پھر check کریں
nslookup tts.yourdomain.com
# یا
dig tts.yourdomain.com
```

### Step 9: SSL Certificate (HTTPS) Setup

```bash
# Certbot install کریں
sudo apt install certbot python3-certbot-nginx -y

# SSL certificate حاصل کریں
sudo certbot --nginx -d tts.yourdomain.com

# Auto-renewal test کریں
sudo certbot renew --dry-run
```

Certbot خودکار طور پر Nginx config update کر دے گا HTTPS کے لیے۔

### Step 10: Firewall Setup

```bash
# UFW firewall enable کریں
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable

# Status چیک کریں
sudo ufw status
```

### Step 11: Testing

**بغیر Domain کے (IP سے):**
```bash
curl http://your-vps-ip/tts/health
```

**Domain کے ساتھ (HTTP):**
```bash
curl http://tts.yourdomain.com/health
```

**Domain کے ساتھ (HTTPS):**
```bash
curl https://tts.yourdomain.com/health
```

**Browser میں:**
```
https://tts.yourdomain.com/health
```

### Step 12: Backend میں URL Update کریں

اپنے main project کی backend میں TTS service URL update کریں:

**Backend .env file:**
```env
TTS_SERVICE_URL=https://tts.yourdomain.com
# یا بغیر domain کے
TTS_SERVICE_URL=http://your-vps-ip/tts
```

---

## 🔧 مفید Commands

### Service Management:
```bash
# Service status
sudo supervisorctl status python-tts

# Service restart
sudo supervisorctl restart python-tts

# Service stop
sudo supervisorctl stop python-tts

# Service start
sudo supervisorctl start python-tts

# Logs دیکھیں
sudo tail -f /var/log/python-tts/out.log
sudo tail -f /var/log/python-tts/err.log
```

### Nginx Management:
```bash
# Nginx status
sudo systemctl status nginx

# Nginx restart
sudo systemctl restart nginx

# Nginx reload (بغیر downtime)
sudo systemctl reload nginx

# Nginx logs
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

### Code Update کریں:
```bash
# Project directory میں جائیں
cd /var/www/ai_voice_caller/python-tts-service

# Git pull کریں
git pull origin main

# Dependencies update (اگر ضرورت ہو)
source venv/bin/activate
pip install -r requirements.txt

# Service restart کریں
sudo supervisorctl restart python-tts
```

---

## 🚨 Troubleshooting

### Problem: Service start نہیں ہو رہی

```bash
# Logs check کریں
sudo tail -n 50 /var/log/python-tts/err.log

# Manually run کر کے error دیکھیں
cd /var/www/ai_voice_caller/python-tts-service
source venv/bin/activate
python app.py
```

### Problem: 502 Bad Gateway

```bash
# Service چل رہی ہے check کریں
sudo supervisorctl status python-tts

# Port 5001 listen کر رہا ہے check کریں
sudo netstat -tulpn | grep 5001

# Service restart کریں
sudo supervisorctl restart python-tts
```

### Problem: Permission Errors

```bash
# Ownership fix کریں
sudo chown -R root:root /var/www/ai_voice_caller/python-tts-service
sudo chmod -R 755 /var/www/ai_voice_caller/python-tts-service

# Directories writable بنائیں
sudo chmod -R 777 /var/www/ai_voice_caller/python-tts-service/generated_audio
sudo chmod -R 777 /var/www/ai_voice_caller/python-tts-service/voice_samples
```

### Problem: CORS Errors

Nginx config میں CORS headers شامل ہیں (Step 7 دیکھیں)۔ اگر پھر بھی issue ہے:

```bash
# Nginx config check کریں
sudo nginx -t

# Nginx reload کریں
sudo systemctl reload nginx
```

---

## 📊 Performance Optimization

### 1. Increase Worker Processes

**app.py کو update کریں** (production کے لیے):

```python
if __name__ == '__main__':
    port = int(os.getenv('PORT', 5001))
    app.run(
        host='0.0.0.0',
        port=port,
        debug=False,  # Production میں False
        threaded=True  # Multiple requests handle کرے
    )
```

### 2. Gunicorn استعمال کریں (Better Performance)

```bash
# Gunicorn install کریں
source venv/bin/activate
pip install gunicorn

# Supervisor config update کریں
sudo nano /etc/supervisor/conf.d/python-tts.conf
```

**Updated config:**
```ini
[program:python-tts]
directory=/var/www/ai_voice_caller/python-tts-service
command=/var/www/ai_voice_caller/python-tts-service/venv/bin/gunicorn -w 4 -b 0.0.0.0:5001 app:app
user=root
autostart=true
autorestart=true
stderr_logfile=/var/log/python-tts/err.log
stdout_logfile=/var/log/python-tts/out.log
```

```bash
# Reload supervisor
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl restart python-tts
```

---

## ✅ Final Checklist

- [ ] VPS پر Python اور dependencies installed
- [ ] Project clone اور virtual environment setup
- [ ] Dependencies install (requirements.txt)
- [ ] .env file configured
- [ ] Service manually test کی
- [ ] Supervisor configured اور service running
- [ ] Nginx reverse proxy configured
- [ ] Domain DNS configured (اختیاری)
- [ ] SSL certificate installed (اختیاری)
- [ ] Firewall rules set
- [ ] Backend میں TTS URL updated
- [ ] Production testing complete

---

## 🎯 تجویز کردہ Setup

**بہترین practice:**
1. ✅ Domain استعمال کریں (e.g., `tts.yourdomain.com`)
2. ✅ SSL Certificate لگائیں (HTTPS)
3. ✅ Gunicorn استعمال کریں (Performance)
4. ✅ Regular backups لیں
5. ✅ Monitoring setup کریں

**Minimum Setup (Development/Testing):**
1. Supervisor سے service run کریں
2. Nginx reverse proxy
3. IP address سے access کریں

---

## 📞 URLs کی مثالیں

### Development (Local):
```
http://localhost:5001/health
http://localhost:5001/api/voices/test
```

### Production (IP only):
```
http://123.45.67.89/tts/health
http://123.45.67.89/tts/api/voices/test
```

### Production (Domain with SSL):
```
https://tts.yourdomain.com/health
https://tts.yourdomain.com/api/voices/test
```

---

**یاد رکھیں:** Domain setup اختیاری ہے لیکن production کے لیے بہتر ہے۔ آپ شروع میں IP سے test کر سکتے ہیں، پھر بعد میں domain add کر سکتے ہیں۔
