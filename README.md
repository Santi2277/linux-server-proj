# Linux Server Configuration - Udacity

### Full Stack Web Development ND

_______________________

## About

In this project an Amazon Lightsail Linux server instance will be created, in  
which the python web server Item Catalog will be running (2nd Udacity project).

The Linux server instance is previously configured to make it secure. Some  
modifications will have to be done to the Item Catalog server too.

The steps to achieve these goals are progressively explained on this document.

Instance public IP is 18.130.251.208 and SSH port is 2200. Access by url:  
http://18.130.251.208/


## Prerequisites

* [VirtualBox 3](https://www.virtualbox.org/wiki/Download_Old_Builds_5_1)  
Necessary download and install to use vagrant on it

* [Vagrant](https://www.vagrantup.com/downloads.html)  
Install and continue reading to get it running


## Installation

Download and install all of the prerequisites shown above.  

To initialize your Vagrant Virtual Machine choose a directory to  
run `vagrant init` which marks it as the root directory and creates  
configuration files (such as VagrantFile). Run `vagrant up` to initialize  
it. Then run `vagrant ssh` to log in.


## Steps

### 1) Get your server

 - Create an AWS account (if you don't have one)

 - Create a Linux Ubuntu (OS Only) instance on Amazon Lightsail

### 2) Connect to your server with Vagrant (your own SSH client)

 - Download your private key (account->SSH keys)

 - Copy it to /home/vagrant/.ssh (I used `sudo nano keyfilenamehere` inside .shh)

 - Change permission (only owner can read/write): `chmod 600 keyfilenamehere`

 - Connnect (ssh): `ssh -i /home/vagrant/.ssh/keyfilenamehere ubuntu@publicIPhere`

### 3) Update installed packages

 - Update packages: `sudo apt-get update`

 - Upgrade packages: `sudo apt-get upgrade`

### 4) Change SSH port from 22 to 2200

 - Configure Lightsail firewall: on instance->networking->firewall add another rule  
(Custom, TCP, 2200)

 - Change port: `sudo nano /etc/ssh/sshd_config` change Port 22 to 2200

 - If there's a System restart required message just reboot instance

 - Connection: now to connect (SSH) especify port too (-p) `ssh -i /home/vagrant/.ssh/keyfilenamehere  
-p 2200 ubuntu@publicIPhere`

### 5) Configure  Uncomplicated Firewall (UFW) to allow SSH (port 2200), HTTP (port 80),  
### and NTP (port 123) connections

 - Check that it begins inactive: `sudo ufw status`

 - Deny incoming connections: `sudo ufw default deny incoming`

 - Allow outgoing requests: `sudo ufw default allow outgoing`

 - Allow ssh: `sudo ufw allow ssh`

 - Allow tcp connections from port 2200: `sudo ufw allow 2200/tcp`

 - Allow tcp connections from port 80: `sudo ufw allow 80/tcp`

 - Allow tcp connections from port 123: `sudo ufw allow 123/tcp`

 - Enable: `sudo ufw enable`

### 6) Give grader access

 - Add user: `sudo adduser grader`

 - Give grader permission to sudo: `sudo visudo` to edit sudoers file, add line  
`grader ALL=(ALL:ALL) ALL` after line `root ALL=(ALL:ALL) ALL`

 - Check: you can change user to check it `sudo su - grader`

### 7) Create an SSH key for grader

(to connect with it instead of the password)

 - Create key-pair (from ubuntu user): `ssh-keygen`

 - Create folder (in grader user): `mkdir .ssh`

 - Create file: `touch .ssh/authorized_keys`

 - Copy public key (.pub) content to it: `nano .ssh authorized_keys`

 - Connect with key-pair from ubuntu user: `ssh grader@instancepubliciphere  
-p 2200 -i /home/ubuntu/graderkey`

### 8) Configure the local timezone to UTC

 - run `sudo dpkg-reconfigure tzdata`, select None of the above then UTC

### 9) Install and configure Apache server

 - Install: `sudo apt-get install apache2`

 - Check if it works: access from your browser to the instance public IP
(it serves /var/www/html/index.html)

 - Install mod-wsgi: `sudo apt-get install libapache2-mod-wsgi`
(to make it serve a Python mod-wsgi application server)

 - Configure Apache to handle requests using the WSGI module: run  
`sudo nano /etc/apache2/sites-enabled/000-default.conf` add line above </VirtualHost>  
`WSGIScriptAlias / /var/www/html/myapp.wsgi`

 - Restart Apache: `sudo apache2ctl restart`

 - Check if it works: create file `sudo nano /var/www/html/myapp.wsgi` with the following  
content and connect again
```python
def application(environ, start_response):
    status = '200 OK'
    output = 'Hello Udacity!'

    response_headers = [('Content-type', 'text/plain'), ('Content-Length', str(len(output)))]
    start_response(status, response_headers)

    return [output]
```
### 10) Install and configure Postgres

 - Install: `sudo apt-get install postgresql`

 - Check remote connections aren't allowed: `sudo nano /etc/postgresql/9.5/main/pg_hba.conf`  
(9.5 or your postgresql version)

 - Change user to postgres: `sudo su - postgres`

 - Connect to PostgreSQL shell: `psql`

### 11) New db user named catalog with limited permissions

 - Create db catalog: `CREATE DATABASE catalog;`

 - Create user catalog: `CREATE USER catalog;`

 - Give user catalog limited permissions to catalog db:  
`ALTER ROLE catalog CREATEDB;`

 - Give a password to catalog user: `\password catalog`

 - Check users: `\du`

 - Go out from PostgreSQL shell `\q` then return to ubuntu user 'exit'

### 12) Install Git

 - Install: `sudo apt-get install git`

### 13) Clone Item Catalog

 - Change directory to /var/www (from ubuntu user)

 - Clone repository: `sudo git clone https://github.com/Santi2277/fullstack-nanodegree-vm.git`

 - Get only the important files: `sudo mv /var/www/fullstack-nanodegree-vm/vagrant/catalog /var/www/`  
to get catalog folder and remove the other files `sudo rm -rf fullstack-nanodegree-vm`  
(now you have the project in /var/www/catalog)

 - change the owner: `sudo chown -R ubuntu:ubuntu catalog/`

### 14) Setup needs to make Item Catalog run on the Apache server 

 - run  `sudo nano item-catalog.py` to put the app.run() inside the  
if __name__ == '__main__': block (Flask mod_wsgi watch out)

 - Install Flask: `sudo apt-get install python-pip` then `sudo pip install Flask`

 - In catalog change name of the main app: `mv item-catalog.py __init__.py`

 - In that file change `app.run(host='0.0.0.0', port=8000)` for `app.run()`

 - Install more needed packages:
`sudo pip install httplib2`
`sudo pip install requests`
`sudo pip install --upgrade oauth2client`
`sudo pip install sqlalchemy`
`sudo apt-get install libpq-dev`
`sudo apt-get install psycopg2`

 - Make catalog project available for Apache: create file  
`sudo nano /etc/apache2/sites-available/catalog.conf` with that content  
``` 
  <VirtualHost *:80>
		ServerName XX.XX.XX.XX
		ServerAdmin admin@xx.xx.xx.xx
		WSGIScriptAlias / /var/www/catalog.wsgi
		<Directory /var/www/catalog/>
			Order allow,deny
			Allow from all
			Options -Indexes
		</Directory>
		Alias /static /var/www/catalog/static
		<Directory /var/www/catalog/static/>
			Order allow,deny
			Allow from all
			Options -Indexes
		</Directory>
		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log combined
   </VirtualHost>
```
 - Enable catalog virtual host: `sudo a2ensite catalog`

 - Restart Apache: `sudo service apache2 reload`

 - Create .wsgi file: `sudo nano /var/www/catalog.wsgi` with
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")

from __init__ import app as application
application.secret_key = 'Add your secret key'
```
 - Restart Apache: `sudo service apache2 reload`

 - Edit database previous connections code: ``__init__.py`` and database_setup.py
`engine = create_engine('postgresql://catalog:catalogpasswordhere@localhost/catalog')`
(Parameters follow this structure postgresql://user:password@host_ip:port/database)
(now it's postgresql database and before was an sqlite database)

 - Disable default Apache server: `sudo a2dissite 000-default.conf`

 - Restart Apache: `sudo service apache2 reload`

 - Install: `sudo pip install psycopg2-binary`

 - Setup database: `sudo python database_setup.py`

 - Restart Apache: `sudo service apache2 reload`

 - Now server is running on http://18.130.251.208/

### 15) Make Google Auth work

 - Add to authorized jascript origins `http://18.130.251.208.xip.io`

 - Now server is running on http://18.130.251.208.xip.io


## Installed software summary (and changes)

VirtualBox, Vagrant, update-upgrade ubuntu packages (on instance), Apache (configuration  
changes for mod-wsgi and server run), mod-wsgi, Postgres, git, Flask, Python get modules  
used in Item Catalog server, psycopg2


## Third-party resources that helped this project

https://www.vagrantup.com/  
https://aws.amazon.com/lightsail/  
https://www.udacity.com/  
https://www.digitalocean.com/community/tutorials/how-to-use-ssh-to-connect-to-a-remote-server-in-ubuntu  
https://www.ssh.com/ssh/command/  
https://forums.aws.amazon.com/thread.jspa?threadID=160352    
https://help.ubuntu.com/community/UFW  
https://stackoverflow.com/  
https://www.digitalocean.com/community/tutorials/how-to-add-delete-and-grant-sudo-privileges-to-users-on-a-debian-vps  
https://askubuntu.com/questions/323131/setting-timezone-from-terminal  
https://httpd.apache.org/  
https://www.tutorialspoint.com/postgresql/postgresql_syntax.htm  
http://flask.pocoo.org/docs/1.0/deploying/mod_wsgi/  
https://bogotobogo.com/python/Flask/Python_Flask_HelloWorld_App_with_Apache_WSGI_Ubuntu14.php  
https://www.1and1.com/cloud-community/learn/web-server/server-management/how-to-fix-http-error-code-500-internal-server-error/  
https://stackoverflow.com/questions/36020374/google-permission-denied-to-generate-login-hint-for-target-domain-not-on-localh/43644795   

