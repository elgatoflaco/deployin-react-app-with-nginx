# Deploying REACT App with Nginx

Now we are going to install required softwares

## Nginx setup

To install nginx run below command

`sudo apt-get install nginx -y`

Once the installation get completed, we can check the status.

`sudo systemctl status nginx`

Just press Ctrl + C to close this. If it is not running and status is inactive then we have to start it.

`sudo systemctl start nginx`

Then run this so that Nginx starts on startup:

`sudo systemctl enable nginx`

For more details on nginx, please go through the documentation.
https://nginx.org/en/docs/

## Node.js setup

Now we have to install node.js in our server. We are going to use NVM(Node Version Manager). Let’s start the installations.

sudo apt-get update
Then install the following packages.

sudo apt-get install build-essential libssl-dev
You may be prompted with questions for which you should respond with “Y”. Once completed, we are going to download the NVM install script:
```
cd ~
curl -sL https://raw.githubusercontent.com/creationix/nvm/v0.33.6/install.sh -o install_nvm.sh
bash install_nvm.sh
source ~/.profile
nvm install 8.9.0 //as we are going to use node 8.9.0 LTS
nvm use 8.9.0
node -v
# Outputs: "v8.9.0"
npm -v
# output: "5.5.1"
```

other options is install node

```
curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
sudo apt-get install -y nodejs 
```

## Configurations

Now it’s time to add domain. First we need to configure the Nginx. We have to remove the original default configuration file.

`sudo rm /etc/nginx/sites-available/default`

Then we have to enter our information to default file. We are going to use nano to edit the file. To save and quit Nano, press Ctrl X. Then it will ask if you want to save the file (do Ctrl + O). Otherwise, do Ctrl + X if you do not want to save the changes.

`sudo nano /etc/nginx/sites-available/default`

Copy and paste the following and replace your_domain.comwith your own domain.

```
server {
    listen 80;
    server_name your_domain.com www.your_domain.com;
    location / {
        proxy_pass http://127.0.0.1:3080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_redirect off;
     }
}
```

This will have all HTTP web traffic redirected to port 3080. I’m going to use my dmain https://domains.google/. On your domain provider side, you’ll have to point the DNS to your Google Cloud Platform VM public IP. 

For type “A”, enter “@” in “Host” field. Then enter your IPv4 Public IP of your vm instance to “Points to” field. Then save it.

The last step is reloading Nginx to accept our new configurations.

`sudo systemctl reload nginx`

## Application Setup

1. Create root directory and clone your repos to the server

```
sudo mkdir /var/www
cd /var/www/
```

Since you wouldn’t want to use sudo every time you interact with files in /var/www or /www, let’s give your user privileges to these folders. Constantly using sudo increases the chances of accidentally trashing your system.

```
sudo gpasswd -a "$USER" www-data
sudo chown -R "$USER":www-data /var/www
find /var/www -type f -exec chmod 0660 {} \;
sudo find /var/www -type d -exec chmod 2770 {} \;
```

If you decide to use /www, change the /var/www in those commands to /www.

Now that you are inside either /www or /var/www and have given your user privileges, let’s clone your repository, install dependencies, and build your React app. You can either clone it with HTTPS or set up a deploy key and clone it via SSH.

```
cd ~/
git clone REPO_PATH
```
Note*- It will ask you for the user name and password of your git account whenever you will do any git operation. If you don’t want to enter user name and password every time, then you have to fork it.

2. Install the npm libraries

```
cd ./REPO_NAME
npm install --production
```
It will install all the libraries that we defined under “dependencies” in our package.json file. We will use these libraries to run the server files. We are not going to install any libraries that we defined in “devDependencies”. Because we already have bundle.js built for the production.

3. Start the application in the production

I have the below command in our package.json

```
NODE_ENV=production node server.js
```

Once application get started, you can enter your domain url in browser. It will serve your application.

4. Start application with pm2

But once you hit Crtl + C, your web application will stop. This doesn’t work as an option because we need it always running. We will use pm2 to keep it running. Install with:

`npm install pm2 -g`

You may need to use sudo because we are going install pm2 globally.

Then we need to add one pm2 config file to our application.
http://pm2.keymetrics.io/docs/usage/application-declaration/

ecosystem.config.js

```
module.exports = {
  apps : [
      {
        name: "<your application name>",
        script: "./<path to>/server.js",
        watch: true,
        env: {
            "PORT": 8080,//you can choose
            "NODE_ENV": "development"
        },
        env_production: {
            "PORT": 3000,//you can choose
            "NODE_ENV": "production",
        }
      }
  ]
}
```

It is good to place this config file in the root directory of your project.

Then run the below command.

`pm2 start ecosystem.config.js --env production`

It will start your application in production mode. To see the information about this process, you have to run the below command.

`pm2 show <app name>`

To stop and start the server, you can run:

`pm2 stop <app name>`
`pm2 start <app name>`

## SSL Setup

Now we are going to setup SSL. We will be using Letsencrypt. For this you need a domain name and it is mandatory to give a domain name while generating key files for SSL.

You can follow the steps given in digitalocean.
https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04

```
cd ~
sudo add-apt-repository ppa:certbot/certbot
# This is the PPA for packages prepared by Debian Let's Encrypt Team and backported for Ubuntu(s).
# More info: https://launchpad.net/~certbot/+archive/ubuntu/certbot
# Press [ENTER] to continue or ctrl-c to cancel adding it
sudo apt-get update
sudo apt-get install python-certbot-nginx
# Y
sudo ufw status
# Status: inactive
sudo ufw allow 'Nginx Full'
# Make sure the domain is pointing to the server at this point.
sudo certbot --nginx -d <your_domain.com>-d <www.your_domain.com>
# Enter email
# A to agree
# Share email so Y/N
# Fails if no domain
# otherwise 1 or 2 for redirect of traffic
# I recommend 2
```

