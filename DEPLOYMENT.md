# MyPetBruno Deployment Guide

## Docker + Self-Hosted Deployment

### Prerequisites
- Docker & Docker Compose installed
- VPS/Server running Linux (Ubuntu 20.04+ recommended)
- Domain name (optional, for HTTPS)
- M-Pesa Daraja credentials

---

## Local Development with Docker

```bash
# Clone repository
git clone https://github.com/metalinktechnologies/MyPetBruno1.git
cd MyPetBruno1

# Create .env file
cp .env.example .env
# Edit .env and fill in your M-Pesa credentials

# Start the app
docker-compose up -d

# Check logs
docker-compose logs -f web

# Stop
docker-compose down
```

App will be available at `http://localhost:5000`

---

## Production Deployment on VPS (AWS, GCP, DigitalOcean, etc.)

### 1. Connect to Your VPS

```bash
ssh root@your_server_ip
```

### 2. Install Docker & Docker Compose

```bash
# Update system
apt update && apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Install Docker Compose
apt install docker-compose -y

# Start Docker daemon
systemctl start docker
systemctl enable docker
```

### 3. Clone Repository

```bash
cd /opt
git clone https://github.com/metalinktechnologies/MyPetBruno1.git mypetbruno
cd mypetbruno
```

### 4. Configure Environment

```bash
cp .env.example .env
nano .env
# Fill in your M-Pesa credentials and set:
# MPESA_ENV=production (when ready)
# MPESA_CALLBACK_URL=https://yourdomain.com/api/mpesa/callback
```

### 5. Start the Application

```bash
docker-compose up -d
docker-compose logs -f
```

Verify it's running:
```bash
curl http://localhost:5000
```

---

## Nginx Reverse Proxy + SSL (Recommended)

### Install Nginx

```bash
apt install nginx -y
systemctl start nginx
systemctl enable nginx
```

### Create Nginx Config

Create `/etc/nginx/sites-available/mypetbruno`:

```nginx
upstream mypetbruno {
    server 127.0.0.1:5000;
}

server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    
    client_max_body_size 10M;

    location / {
        proxy_pass http://mypetbruno;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
    }
}
```

Enable it:
```bash
ln -s /etc/nginx/sites-available/mypetbruno /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

### Setup HTTPS with Let's Encrypt

```bash
apt install certbot python3-certbot-nginx -y
certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

---

## Database Persistence

SQLite database is stored in the `data/` volume. To back it up:

```bash
# Copy database locally
docker-compose exec web cp /app/mypetbruno.db /app/data/backup.db
docker cp mypetbruno:/app/data/backup.db ./backup.db

# Restore from backup
docker cp ./backup.db mypetbruno:/app/data/mypetbruno.db
```

---

## Managing the Application

### View Logs
```bash
docker-compose logs -f web
```

### Restart
```bash
docker-compose restart
```

### Update Code
```bash
git pull
docker-compose up -d --build
```

### Stop
```bash
docker-compose down
```

---

## Monitoring & Maintenance

### Health Check
```bash
curl http://localhost:5000/
```

### Database Maintenance
For production, consider migrating to PostgreSQL:

```bash
# Add to docker-compose.yml
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: mypetbruno
      POSTGRES_PASSWORD: secure_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
```

Then update `app.py` to use SQLAlchemy with PostgreSQL.

---

## Troubleshooting

### Container won't start
```bash
docker-compose logs web
```

### Port already in use
```bash
sudo lsof -i :5000
kill -9 <PID>
```

### M-Pesa callback not working
- Ensure `MPESA_CALLBACK_URL` is publicly accessible
- Test: `curl -X POST https://yourdomain.com/api/mpesa/callback -H "Content-Type: application/json" -d '{}'`
- Check firewall rules

### SSL certificate renewal
```bash
certbot renew --dry-run
certbot renew
```

---

## Security Checklist

- [ ] Change all M-Pesa credentials in `.env`
- [ ] Set `MPESA_ENV=production` when ready
- [ ] Enable HTTPS with SSL
- [ ] Use strong passwords for any services
- [ ] Restrict SSH access (use SSH keys, not passwords)
- [ ] Enable firewall rules
- [ ] Set up log rotation
- [ ] Regular database backups
- [ ] Monitor server resources

---

## Support

For issues, check:
- Application logs: `docker-compose logs -f`
- M-Pesa Daraja docs: https://developer.safaricom.co.ke
- Nginx error logs: `/var/log/nginx/error.log`
