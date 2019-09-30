#Odoo deployment on docker with postgresql DB and nginx reverse proxy

##Scenario:
We are going to deploy every service separately on different servers.

1.Server1 is for Postgres Database (192,168.1.10).

2.Server2 is for Odoo Application (192.168.1.11).

3.Server3 is for Nginx as (Reverse Proxy - Caching - Loadbalancer) (192.168.1.12).

The approach we are going to use is Active-Active implementation.

Preassuming all servers have Ubuntu 16.04 server installed on them.


#Postgres Installation:

##Setup:
All servers have to be running & Ubuntu server 16.04 installed on them.

Important Note: before running installation scripts, you have to edit script parameters specified in it.
Step 1 (Master DB):
```console
sudo wget https://github.com/somud17/Odoo-Deploy/tree/master/Proj_dedicatedservers/master-install.sh
sudo chmod +x master-install.sh
sudo nano master-install.sh
ODOO_DB_USER="odoo" # DB user going to be used by Odoo
ODOO_DB_PASS="odoo"
MASTER_IP="192.168.1.10" # Master Server IP address
SLAVE_IP="192.168.1.11" # Slave Server IP address
SLAVE_USER="dave" # SSH user on Slave Server for transfering key files
SLAVE_PASS="dave"
NETWORK="192.168.1.0/24" # Network IP & Subnet
```

After modifying script parameters to suit your network setup, save modification by pressing Ctrl+O then Enter then Ctrl+X to exit the editor.

Then run the script to install:
```console
sudo ./master-install.sh
```
Basically, the script will install PostgreSQL and repmgr then configure both of them in addition to generating & transferring password-less SSH key 
to Slave Server and creating ready scripts in the home directory to help DB admin to promote or demote server easily.

The password-less key file will be used by both PostgreSQL servers to sync data together without manual authenticating.



##Step 2 (Slave DB):
Download second script on Slave server to Install PostgreSQL & Password-less SSH keys:
```console
sudo wget https://github.com/somud17/Odoo-Deploy/tree/master/Proj_dedicatedservers/slave-install.sh
sudo chmod +x slave-install.sh
sudo nano slave-install.sh
MASTER_IP="192.168.1.10" # Enter Master server IP address
SLAVE_IP="192.168.1.11" # Enter Slave server IP address
```

Then, save modification by pressing Ctrl+O then Enter then Ctrl+X to exit the editor.

It will install PostgreSQL, repmgr & password-less keys and clones Master server configurations.



##Step 3 (PgBouncer):
Download final script to install PgBouncer server.
```console 
sudo wget https://github.com/somud17/Odoo-Deploy/tree/master/Proj_dedicatedservers/pgbouncer-install.sh
sudo chmod +x pgbouncer-install.sh
sudo nano pgbouncer-install.sh
MASTER_IP="192.168.1.10" # Master server IP address
SLAVE_IP="192.168.1.11" # Slave server IP address
ODOO_DB_USER="odoo" # DB user going to be used by Odoo
ODOO_DB_PASS="odoo"
```
Then, save modification by pressing Ctrl+O then Enter then Ctrl+X to exit the editor.

##Nginx Installation:
Log in to Nginx server (192.168.1.12), download Nginx install script:

sudo wget https://github.com/somud17/Odoo-Deploy/tree/master/Proj_dedicatedservers/nginx-install.sh
sudo chmod +x nginx-install.sh
Let's modify Nginx installation script before execution.
```console
sudo nano nginx-install.sh
OE_DOMAIN="odoo.example.info *.odoo.example.info" # Modify it to your own domain name it's important for SSL registering
OE_HOST="192.168.1.11" # Change it to Odoo server Local IP address
OE_PORT="8069"
NGINX_CONFIG="odoo"
NGINX_CONFIG_PATH="/etc/nginx/sites-available/${NGINX_CONFIG}"
```

Then, save modification by pressing Ctrl+O then Enter then Ctrl+X to exit the editor.

Run the script:
```
sudo ./nginx-install.sh
```
Then, let's review our website configuration file.
```
sudo nano /etc/nginx/sites-enabled/odoo
```
We will find the following:
```consle
#odoo server
upstream odoo {
        server 192.168.1.11:8069 weight=1 fail_timeout=0;
        # For more instances of Odoo add them below
        #server <SECOND-SERVER>:8069 weight=1 fail_timeout=0;
}
upstream odoochat {
        server 192.168.1.11:8072 weight=1 fail_timeout=0;
        # For more instances of Odoo add them below
        #server <SECOND-SERVER>:8072 weight=1 fail_timeout=0;
}

# http -> https
#server {
#        listen 80;
#        listen [::]:80 ipv6only=on;
#        server_name odoo.example.info *.odoo.example.info;
#        add_header Strict-Transport-Security max-age=2592000;
#        rewrite ^/.*$ https://\$host\$request_uri? permanent;
#}

server {
        #listen 443;
        #listen [::]:443 ipv6only=on;
        listen 80;
        listen [::]:80 ipv6only=on;
        server_name odoo.example.info *.odoo.example.info;
        proxy_read_timeout 720s;
        proxy_connect_timeout 720s;
        proxy_send_timeout 720s;
        keepalive_timeout 60;

        # Add Headers for odoo proxy mode
        proxy_set_header X-Forwarded-Host \$host;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
        proxy_set_header X-Real-IP \$remote_addr;

        # SSL parameters
        #ssl on;
        #ssl_certificate /etc/letsencrypt/live/odoo.example.info/fullchain.pem;
        #ssl_certificate_key /etc/letsencrypt/live/odoo.example.info/privkey.pem;
        #ssl_session_timeout 30m;
        #ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        #ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES1$
        #ssl_prefer_server_ciphers on;

        # log
        access_log /var/log/nginx/odoo.access.log;
        error_log /var/log/nginx/odoo.error.log;

        # Redirect requests to odoo backend server
        location / {
                proxy_redirect off;
                proxy_pass http://odoo;
        }

        location /longpolling {
                proxy_pass http://odoochat;
        }

        # cache some static data in memory for 60mins.
        # under heavy load this should relieve stress on the OpenERP web interface a bit.
        location ~* /[0-9a-zA-Z_]*/static/ {
                proxy_cache_valid 200 60m;
                proxy_buffering on;
                expires 864000;
                proxy_pass http://odoo;
        }

        # common gzip
        gzip_types text/css text/less text/plain text/xml application/xml application/json application/javascript;
        gzip on;
}
```
Then, save modification by pressing Ctrl+O then Enter then Ctrl+X to exit the editor.

Important note: don't enable SSL section yet, we need first to start service without encryption to register SSL certificate then we reconfigure 
SSL section and link nginx with private & public key files.

Restart Nginx service by:
```
sudo service nginx restart
```
After that we start registering SSL certificates by Certbot:
```
sudo certbot --nginx certonly
```
You will be prompt to enter email & company's information to be included in the certificate, in addition, you will be notified on the specified 
email whenever certificate requires renewal.

After filling all information needed, choose website domain you want to create a certificate for it. In our case, it's odoo.example.info

Lastly, it will show you where on the system the keys are stored keep it in mind because we are going to include it in the configuration file.

After that we head back to our website configuration file again:

```console
sudo nano /etc/nginx/sites-enabled/odoo
#odoo server
upstream odoo {
        server 192.168.1.11:8069 weight=1 fail_timeout=0;
        #server <SECOND-SERVER>:8069 weight=1 fail_timeout=0;
}
upstream odoochat {
        server 192.168.1.11:8072 weight=1 fail_timeout=0;
        #server <SECOND-SERVER>:8072 weight=1 fail_timeout=0;
}

# http -> https
server {
        listen 80;
        listen [::]:80 ipv6only=on;
        server_name odoo.example.info *.odoo.example.info;
        add_header Strict-Transport-Security max-age=2592000;
        rewrite ^/.*$ https://\$host\$request_uri? permanent;
}

server {
        listen 443;
        listen [::]:443 ipv6only=on;
        #listen 80;
        #listen [::]:80 ipv6only=on;
        server_name odoo.example.info *.odoo.example.info;
        proxy_read_timeout 720s;
        proxy_connect_timeout 720s;
        proxy_send_timeout 720s;
        keepalive_timeout 60;

        # Add Headers for odoo proxy mode
        proxy_set_header X-Forwarded-Host \$host;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
        proxy_set_header X-Real-IP \$remote_addr;

        # SSL parameters
        ssl on;
        ssl_certificate /etc/letsencrypt/live/odoo.example.info/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/odoo.example.info/privkey.pem;
        ssl_session_timeout 30m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES1$
        ssl_prefer_server_ciphers on;

        # log
        access_log /var/log/nginx/odoo.access.log;
        error_log /var/log/nginx/odoo.error.log;

        # Redirect requests to odoo backend server
        location / {
                proxy_redirect off;
                proxy_pass http://odoo;
        }

        location /longpolling {
                proxy_pass http://odoochat;
        }

        # cache some static data in memory for 60mins.
        # under heavy load this should relieve stress on the OpenERP web interface a bit.
        location ~* /[0-9a-zA-Z_]*/static/ {
                proxy_cache_valid 200 60m;
                proxy_buffering on;
                expires 864000;
                proxy_pass http://odoo;
        }

        # common gzip
        gzip_types text/css text/less text/plain text/xml application/xml application/json application/javascript;
        gzip on;
}
```
To configure CertBot to auto-renew certificate periodically, add a scheduled action command in Ubuntu to do it for you:
You will be prompt to choose your favorite text editor for below crontab, choose nano for the sake of easiness.
```console
sudo crontab -e
# Edit this file to introduce tasks to be run by cron.
#
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
#
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').#
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
#
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
#
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
#
# For more information see the manual pages of crontab(5) and cron(8)
#
# m h dom mon dow command
0 3 * * 1 certbot renew --dry-run # Certificate will be renewed everyweek at 3 AM
```

Then, save modification by pressing Ctrl+O then Enter then Ctrl+X to exit the editor.

##Final Step (Odoo): 
If you already have an Odoo server deployed, open Odoo configuration file and modify DB settings:
```
db_host = 192.168.1.12 # PgBouncer server IP
db_port = 6432 # PgBouncer port
db_user = odoo # DB user
db_password = odoo # DB user's password
```

Important Notes:
> On both PostgreSQL servers, you should find 2 scripts promote-server & demote-server to help you change the role of the servers easily in case 
of failure. On PgBouncer server, you should find 2 scripts also switch-node1 & switch-node2 to help you switch easily between Nodes.


###Master Failover:
In case the Master DB failover, login to the Slave server and promote it with promote-server script & then login to PgBouncer server and 
run switch-node# script in order to switch to the new Master (Promoted) DB.

After fixing issues with the old Master server, run demote-server script to change the role of it to be the new Slave.

Finally, everything should be configured right and ready to test.
