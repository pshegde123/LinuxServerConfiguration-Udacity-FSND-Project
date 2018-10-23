# LinuxServerConfigureation
This linux server configuration project is Udacity Full Stack Web Development Nanodegree course project.
Under this project, a user will take a baseline installation of a Linux server and configure the server to actually host a web application.
Here are the details of the server :
* Server IP address : 35.174.190.27 (hosted on Amazon LightSail)
* SSH port : 2200
* Application URL : http://35.174.190.27.xip.io

## Walkthrough
1. __Download and set up PuTTY to connect using SSH in Amazon Lightsail__
Follow the steps mentioned on [ Amazon LightSail ] ( https://lightsail.aws.amazon.com/ls/docs/en/articles/lightsail-how-to-set-up-putty-to-connect-using-ssh ).
Connect the server using command ` ssh ubuntu@35.174.190.27 -i .ssh/id_rsa `.
2. __Create new user named grader and give it the permission to sudo__
Run ` $ sudo adduser grader ` to create a new user named _grader_
Create a new file in the sudoers directory with ` $sudo nano /etc/sudoers.d/grader `
Add the following text ` grader ALL=(ALL) NOPASSWD:ALL `
3.__Enable key based authentication__
  3.1. Generate key pair on your local machine using command ` $ssh-keygen `
  3.2. Log in to linux server ubuntu@35.174.190.27, change user to 'grader' using command ` $su - grader `
  3.3. Copy the public key from step 3.1 to /home/grader/.ssh/authorized_keys
  3.4 From your local machine connect to server using command ` ssh grader@35.174.190.27 -i .ssh/id_rsa `
  3.5 I have used passphrase _udacity_
4. __Update all currently installed packages__
Download package lists with ` $sudo apt-get update `
Fetch new versions of packages with ` $sudo apt-get upgrade `
5. __Change SSH port from 22 to 2200__ 
Run ` $sudo nano /etc/ssh/sshd_config `
Change the port from 22 to 2200
restart SSH service by command ` $sudo service ssh restart`
confirm change is working by command `$netstat -ntlp `, it should show post 2200 listening.  
6. __Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)__
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo ufw allow 123/udp
sudo ufw enable
7. __Configure the local timezone to UTC__
Run ` $sudo dpkg-reconfigure tzdata and then choose UTC `
8. __Disable ssh login for root user__
Run ` $sudo nano /etc/ssh/sshd_config`
Change ` PermitRootLogin without-password` line to `PermitRootLogin no`
Restart ssh with `$sudo service ssh restart`
Now you are only able to login using `ssh grader@35.174.190.27 -p 2200 -i .ssh/id_rsa`
9.__Install Apache__
`sudo apt-get install apache2`
10.__Install mod_wsgi__
Run `sudo apt-get install libapache2-mod-wsgi python-dev`
Enable mod_wsgi with sudo a2enmod wsgi
Start the web server with sudo service apache2 start
11. __Clone the Catalog app from Github__
Install git using: sudo apt-get install git
cd /var/www
sudo mkdir catalog
Change owner of the newly created catalog folder sudo chown -R grader:grader catalog
cd /catalog
Clone your project from github git clone https://github.com/rrjoson/udacity-item-catalog.git catalog
12. Create a catalog.wsgi file, then add this inside:
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'supersecretkey'
13. Rename application.py to init.py mv application.py __init__.py
Install virtual environment
Install the virtual environment sudo pip install virtualenv
Create a new virtual environment with sudo virtualenv venv
Activate the virutal environment source venv/bin/activate
Change permissions sudo chmod -R 777 venv
Install Flask and other dependencies
Install pip with sudo apt-get install python-pip
Install Flask pip install Flask
Install other project dependencies sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils
Update path of client_secrets.json file
nano __init__.py
Change client_secrets.json path to /var/www/catalog/catalog/client_secrets.json
Configure and enable a new virtual host
Run this: sudo nano /etc/apache2/sites-available/catalog.conf
Paste this code:
<VirtualHost *:80>
    ServerName 35.167.27.204
    ServerAlias ec2-35-167-27-204.us-west-2.compute.amazonaws.com
    ServerAdmin admin@35.167.27.204
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
Enable the virtual host sudo a2ensite catalog
Install and configure PostgreSQL
sudo apt-get install libpq-dev python-dev
sudo apt-get install postgresql postgresql-contrib
sudo su - postgres
psql
CREATE USER catalog WITH PASSWORD 'password';
ALTER USER catalog CREATEDB;
CREATE DATABASE catalog WITH OWNER catalog;
\c catalog
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO catalog;
\q
exit
Change create engine line in your __init__.py and database_setup.py to: engine = create_engine('postgresql://catalog:password@localhost/catalog')
python /var/www/catalog/catalog/database_setup.py
Make sure no remote connections to the database are allowed. Check if the contents of this file sudo nano /etc/postgresql/9.3/main/pg_hba.conf looks like this:
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
Restart Apache
sudo service apache2 restart
Visit site at http://35.174.190.27

