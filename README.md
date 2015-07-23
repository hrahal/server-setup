# server setup

standards for setting up Ubuntu, node, MongoDB, nginx, pm2 env for stagging and production with one server over an ip address 

### Version
0.0.1

### login as root 

```sh
$ ssh root@ipnumber
```
### users

add users to group
```sh
adduser simon
```

add user to sudo group:
 
```sh   
gpasswd -a simon sudo 
```   

add ssh key to user. on local pc: 

```sh   
cat .ssh/ids_rsa.pub 
```  

if key doesn't exist generate one: 

```sh
ssh-keygen
```

- copy id_rsa.pub key

on the server as root switch user: 

```sh
su - simon
```

create .ssh dir & .ssh/authorized_keys file, paste the ida_rsa.pub there: 

```ss
mkdir .ssh
chmod 700 .ssh
vim .ssh/authorized_keys
```
change premissions: 

```sh
chmod 600 .ssh/authorized_keys
```

restart ssh service to apply changes

```sh
service ssh restart
```

### Firewall basics 

allow ssh:
```sh                                                         
sudo ufw allow ssh
```                                                 

allow specific ports for ssh, http, ssl/tls: 

```sh
sudo ufw allow 4444/tcp
sudo ufw allow 80/tcp
sudo ufw allow 81/tcp
sudo ufw allow 443/tcp
```

show allowed and enable firewall 

```sh
sudo ufw show added
sudo ufw enable
```

### Timezone

configure server timezone

```sh
sudo dpkg-reconfigure tzdata
```

- a menu will open, choose your city

configure NTP to stay in sync with other servers:

```sh
sudo apt-get update
sudo apt-get install ntp
```

### install packages

####nodejs, npm, express, bower: 
```sh
sudo apt-get update
sudo apt-get install nodejs

sudo apt-get install npm

npm install express -g
npm install bower -g
```

####MongoDB

import public key
```sh
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
```

create a list file: 

```sh
echo "deb http://repo.mongodb.org/apt/ubuntu "$(lsb_release -sc)"/mongodb-org/3.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.0.list
```

reload

```sh
sudo apt-get update
```

install stable:

```sh
sudo apt-get install -y mongodb-org
```

start
```sh
sudo service mongod start
```

### nginx configuration

install

```sh
sudo apt-get update
sudo apt-get install nginx
``` 

start

```sh
sudo service nginx start
```

Create the file yourdomain at /etc/nginx/sites-available/:

```sh
vim /etc/nginx/sites-available/yourdomain
```

something like: 

```sh
#the IP(s) on which your node server is running. I chose port 3000 for production and 8000 for stagging.             
                                                                                
# the nginx server instance                                                     
server {                                                                        
    listen 80;                                                                  
    server_name 104.236.241.255;                                                
    access_log /var/log/nginx/production.log;                                   
			                                                                                   # pass the request to the node.js server with the correct headers                               
   location / {                                                                
     proxy_set_header X-Real-IP $remote_addr;                                  
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;              
     proxy_set_header Host $http_host;                                         
     proxy_set_header X-NginX-Proxy true;                                      
                                                                               
     proxy_pass http://127.0.0.1:3000/;                                        
     proxy_redirect off;                                                       
   }                                                                           
                                                                               
}                                                                              
                                                                                  
server {                                                                        
   listen 81;                                                                  
   server_name 104.236.241.255;                                                
   access_log /var/log/nginx/stagging.log;                                     
                                                                               
   location / {                                                                
     proxy_set_header X-Real-IP $remote_addr;                                  
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;              
     proxy_set_header Host $http_host;                                         
     proxy_set_header X-NginX-Proxy true;                                      
                                                                               
     proxy_pass http://127.0.0.1:8000/;                                        
     proxy_redirect off;                                                       
   }                                                                           
}
```

- save and quit

link your file to site-enabled to apply changes:

```sh
cd /etc/nginx/sites-enabled/ 
ln -s /etc/nginx/sites-available/yourdomain yourdomain
```

restart

```sh
sudo /etc/init.d/nginx restart
```

### PM2

one server as root:

```sh
sudo npm install pm2 -g
```

as user (simon):

try:
```sh
pm2 list
```

if error (EACCES, permission denied )
then you have to give permissions to user: 

```sh
sudo chmod -R 777 .pm2
```

change .pm2 ownership

```sh
chown nobody:nogroup -R .path
```

run an app: 

```sh
pm2 start bin/www
```

#### to start on boot:

```sh
sudo env PATH=$PATH:/usr/bin pm2 startup ubuntu -u simon
```


then save processes: 
```sh
pm2 save
```

- now when rebooting system, your app should run on start

#### deploy app remotely:

add ecosystem.json file to your project (example):

```sh
{
    apps: [
        {
            name: "heatinc-stagging",
            script: "bin/www",
            env: {
                NODE_PORT: 8000,
                env: "production"
            }
        }
    ],
    deploy: {
        stagging: {
            user: "simon",
            host: "104.236.241.255",
            ref: "origin/master",
            repo: "git@github.com:Digitiv-Inc/heat-inc.git",
            path: "~/www/stagging",
            "post-deploy": "npm run deploy; export NODE_PORT=8000; pm2 startOrRestart ecosystem.json -f --env production",
            env: {
                NODE_PORT: 8000
            }
        }
    }
}
```

push setup to server (remotely):

```sh
pm2 deploy ecosystem.json stagging setup
```

deploy app on server (remotely): 

```sh
pm2 deploy ecosystem.json stagging
```

- now on server your stagging app should be running
- to check 

```sh
pm2 list
```

