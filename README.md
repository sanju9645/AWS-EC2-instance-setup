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

 #Clone the project repo

#Install Node
To install Node on Ubuntu follow the steps detailed in: 
https://github.com/nodesource/distributions/blob/master/README.md
```bash
curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash -
sudo apt-get install -y nodejs
```

#Install and Configure PM2
We never want to run node directly in production. Instead we want to use a process manager like PM2 to handle running our backend server. PM2 will be responsible for restarting the App if/when it crashes 😁
```bash
sudo npm install pm2 -g
```

Point pm2 to the location of the server.js file so it can start the app. We can add the --name flag to give the process a descriptive name
```bash
pm2 start /home/ubuntu/apps/yelp-app/server/server.js --name yelp-app
```

Configure PM2 to automatically startup the process after a reboot
```bash
ubuntu@ip-172-31-20-1:~$ pm2 startup
[PM2] Init System found: systemd
[PM2] To setup the Startup Script, copy/paste the following command:
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu --hp /home/ubuntu
```

The output above gives you a specific command to run, copy and paste it into the terminal. The command given will be different on your machine depending on the username, so do not copy the output above, instead run the command that is given in your output.
```bash
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu --hp /home/ubuntu
```

Verify that the App is running
```bash
pm2 status
```

After verify App is running, save the current list of processes so that the same processes are started during bootup. If the list of processes ever changes in the future, you'll want to do another pm2 save
```bash
pm2 save
```

#Deploy React Frontend
Navigate to the client directory in our App code and run
```bash
npm run build.
```
This will create a finalized production ready version of our react frontent in directory called build. The build folder is what the NGINX server will be configured to serve.


#Install and Configure NGINX
Install and enable NGINX
```bash
sudo apt install nginx -y
sudo systemctl enable nginx
```

NGINX is a feature-rich webserver that can serve multiple websites/web-apps on one machine. Each website that NGINX is responsible for serving needs to have a seperate server block configured for it.

Navigate to '/etc/nginx/sites-available'
```bash
cd /etc/nginx/sites-available
```

There should be a server block called default
```bash
ubuntu@ip-172-31-20-1:/etc/nginx/sites-available$ ls
default 
```

The default server block is what will be responsible for handling requests that don't match any other server blocks. Right now if you navigate to your server ip, you will see a pretty bland html page that says NGINX is installed. That is the default server block in action.

We will need to configure a new server block for our website. To do that let's create a new file in /etc/nginx/sites-available/ directory. We can call this file whatever you want, but I recommend that you name it the same name as your domain name for your app. In this example my website will be hosted at burnbitsbistro.xyz so I will also name the new file burnbitsbistro.xyz. But instead of creating a brand new file, since most of the configs will be fairly similar to the default server block, I recommend copying the default config.

```bash
cd /etc/nginx/sites-available
sudo cp default burnbitsbistro.xyz
```

open the new server block file burnbitsbistro.xyz and modify it so it matches below:
```bash
server {
        listen 80;
        listen [::]:80;

         root /home/ubuntu/apps/yelp-app/client/build;

        # Add index.php to the list if you are using PHP
        index index.html index.htm index.nginx-debian.html;

        server_name sanjeev.xyz www.sanjeev.xyz;

        location / {
                try_files $uri /index.html;
        }

         location /api {
            proxy_pass http://localhost:3001;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }

}
```
Let's go over what each line does

The first two lines listen 80 and listen [::]:80; tell nginx to listen for traffic on port 80 which is the default port for http traffic. Note that I removed the default_server keyword on these lines. If you want this server block to be the default then keep it in

root /home/ubuntu/apps/yelp-app/client/build; tells nginx the path to the index.html file it will server. Here we passed the path to the build directory in our react app code. This directory has the finalized html/js/css files for the frontend.

server_name sanjeev.xyz www.sanjeev.xyz; tells nginx what domain names it should listen for. Make sure to replace this with your specific domains. If you don't have a domain then you can put the ip address of your ubuntu server.

The configuration block below is needed due to the fact that React is a Singe-Page-App. So if a user directly goes to a url that is not the root url like https://sanjeev.xyz/restaurants/4 you will get a 404 cause NGINX has not been configured to handle any path ohter than the /. This config block tells nginx to redirect everything back to the / path so that react can then handle the routing.
```bash
        location / {
                try_files $uri /index.html;
        }
```
The last section is so that nginx can handle traffic destined to the backend. Notice the location is for /api. So any url with a path of /api will automatically follow the instructions associated with this config block. The first line in the config block proxy_pass http://localhost:3001; tells nginx to redirect it to the localhost on port 3001 which is the port that our backend process is running on. This is how traffic gets forwarded to the Node backend. If you are using a different port, make sure to update that in this line.

Enable the new site
```bash
sudo ln -s /etc/nginx/sites-available/sanjeev.xyz /etc/nginx/sites-enabled/
systemctl restart nginx
```

#Configure Environment Variables

We now need to make sure that all of the proper environment variables are setup on our production Ubuntu Server. In our development environment, we made use of a package called dotenv to load up environment variables. In the production environment the environment variables are going to be set on the OS instead of within Node.

Create a file called .env in /home/ubuntu/. The file does not need to be named .env and it does not need to be stored in /home/ubuntu, these were just the name/location of my choosing. The only thing I recommend avoid doing is placing the file in the same directory as the app code as we want to make sure we don't accidentally check our environment variables into git and end up exposing our credentials.

Within the .env file paste all the required environment variables
```bash
PORT=3001
PGUSER=postgres
PGHOST=localhost
PGPASSWORD=password123
PGDATABASE=yelp
PGPORT=5432 
NODE_ENV=production
```

You'll notice I also set NODE_ENV=production. Although its not required for this example project, it is common practice. For man other projects(depending on how the backend is setup) they may require this to be set in a production environment.

In the /home/ubuntu/.profile add the following line to the bottom of the file
```bash
set -o allexport; source /home/ubuntu/.env; set +o allexport
```

For these changes to take affect, close the current terminal session and open a new one.

Verify that the environment variables are set by running the printenv
```bash
# printenv
```

#Enable Firewall
```bash
sudo ufw status
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo ufw enable
sudo ufw status
```

#Enable SSL with Let's Encrypt
Nowadays almost all websites use HTTPS exclusively. Let's use Let's Encrypt to generate SSL certificates and also configure NGINX to use these certificates and redirect http traffic to HTTPS.

The step by step procedure is listed at: https://certbot.eff.org/lets-encrypt/ubuntufocal-nginx.html

Install Certbot
```bash
sudo snap install --classic certbot
```


Prepare the Certbot command
```bash
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

Get and install certificates using interactive prompt
```bash
sudo certbot --nginx
```


## Extras
```bash
Install nodejs on project folder

curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash -
sudo apt-get install -y nodejs

node -v
sudo apy install npm

npm i


pm2 start /var/www/html/burnbitsbistro/nudge/backend/server.js

pm2 stop 0

pm2 status

pm2 delete 0

pm2 start /var/www/html/burnbitsbistro/nudge/backend/server.js --name nudge

pm2 startup

pm2 save

pm2 restart nudge

sudo reboot

cd frontend/
npm i
npm run build

install nginx


sudo systemctl status nginx

sudo nginx -t

cd /etc/nginx/sites-available

sudo cp default burnbitsbistro.xyz

edit the config file as below

then 

sudo ln -s /etc/nginx/sites-available/burnbitsbistro.xyz /etc/nginx/sites-enabled/

systemctl restart nginx

set -o allexport; source /var/www/html/burnbitsbistro/nudge/.env; set +o allexport
```

```bash
server {

	root /var/www/html/burnbitsbistro/nudge/frontend/dist;

	# Add index.php to the list if you are using PHP
	index index.html index.htm index.nginx-debian.html;

	server_name burnbitsbistro.xyz www.burnbitsbistro.xyz 3.108.18.241;

	location / {
		try_files $uri /index.html;
	}
	
	location /api {
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_pass http://localhost:3000;
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection 'upgrade';
		proxy_set_header Host $host;
		proxy_cache_bypass $http_upgrade;
	}

    listen [::]:443 ssl ipv6only=on; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/burnbitsbistro.xyz/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/burnbitsbistro.xyz/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot


}
server {
    if ($host = www.burnbitsbistro.xyz) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    if ($host = burnbitsbistro.xyz) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


	listen 80;
	listen [::]:80;

	server_name burnbitsbistro.xyz www.burnbitsbistro.xyz 3.108.18.241;
    return 404; # managed by Certbot




}

```


