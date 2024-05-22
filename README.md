# AWS-EC2-instance-setup
ec2 instance + Nginx + letsencrypt free ssl certificate


---

**Launching a New Instance for Nudge**

---

Launch a new instance with AWS Ubuntu.

1. Create or associate a security group to manage traffic (port 22, 80, 443).
2. SSH into the instance:
   ```bash
   sudo -i
   apt update
   apt install nginx
   nginx -v
   ```

To check the public IP:

```bash
curl ifconfig.me
```

Configure NGINX:

```bash
cd /etc/nginx/conf.d
vi burnbitsbistro.conf

# Example NGINX configuration
server {
  listen 80;
  server_name burnbitsbistro.xyz;
  client_max_body_size 100M;
  index index.html index.php;
  root /var/www/html/burnbitsbistro/;
  access_log /var/log/nginx/burnbitsbistro/access.log;
  error_log /var/log/nginx/burnbitsbistro/error.log;
}

mkdir /var/www/html/burnbitsbistro
mkdir /var/log/nginx/burnbitsbistro
nginx -t  # Test the configuration
systemctl restart nginx  # Restart NGINX
```

Install SSL:

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d burnbitsbistro.xyz
cat /etc/nginx/conf.d/burnbitsbistro.conf  # Verify SSL configuration
```
