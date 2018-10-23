# LinuxServerConfiguration
This is Udacity Full Stack Web Development Nanodegree course project. Purpose of the project is to configure a Linux server to host a web app securely.

Here are the details of the server (hosted on Amazon LightSail):
* Server IP address : 35.174.190.27
* SSH port : 2200
* Application URL : http://35.174.190.27.xip.io

## Configuration changes
1. __Download and set up PuTTY to connect using SSH in Amazon Lightsail__
Follow the steps mentioned on [ Amazon LightSail ] ( https://lightsail.aws.amazon.com/ls/docs/en/articles/lightsail-how-to-set-up-putty-to-connect-using-ssh ) and connect to the server.

2. __Create new user named 'grader' and give it the permission to sudo__
Run ` $ sudo adduser grader ` to create a new user named _grader_
Create a new file in the sudoers directory with ` $sudo nano /etc/sudoers.d/grader `
Save following text in to the file` grader ALL=(ALL) NOPASSWD:ALL `

3.__Enable key based authentication__
  3.1. Generate key pair on your local machine using command ` $ssh-keygen `
        * I have used passphrase as _udacity_
  3.2. Log in to linux server as ubuntu@35.174.190.27, change user to 'grader' using command ` $su - grader `
  3.3. Execute following commands:
        `$mkdir /home/grader/.ssh
         $chown grader:grader /home/grader/.ssh
         $chmod 700 /home/grader/.ssh
         Copy the public key generate in step 3.1 to /home/grader/.ssh/authorized_keys
         $chown grader:grader /home/grader/.ssh/authorized_keys
         $chmod 644 /home/grader/.ssh/authorized_keys ` 
  3.4 From your local machine connect to server using command ` ssh grader@35.174.190.27 -i .ssh/id_rsa `
  3.5 Enter passphrase _udacity_

4. __Update all currently installed packages__
Download package lists with ` $sudo apt-get update `
Fetch new versions of packages with ` $sudo apt-get upgrade `

5. __Change SSH port from 22 to 2200__ 
Run ` $sudo nano /etc/ssh/sshd_config `
Change the port from 22 to 2200
restart SSH service by command ` $sudo service ssh restart`
confirm change is working by command `$netstat -ntlp `, it should show post 2200 listening.  

6. __Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)__
By default, block all incoming connections on all ports:
` $sudo ufw default deny incoming`
Allow outgoing connection on all ports:
`$sudo ufw default allow outgoing`
Allow incoming connection for SSH on port 2200:
`$sudo ufw allow 2200/tcp`
Allow incoming connections for HTTP on port 80:
`$sudo ufw allow www`
Allow incoming connection for NTP on port 123:
`$sudo ufw allow ntp`
To enable the firewall, use:
`$sudo ufw enable`
To check the status of the firewall, use:
`$sudo ufw status`

7. __Configure the local timezone to UTC__
Run ` $sudo timedatectl set-timezone UTC `

8. __Disable ssh login for root user__
Run ` $sudo nano /etc/ssh/sshd_config`
Change ` PermitRootLogin without-password` line to `PermitRootLogin No`
Restart ssh with `$sudo service ssh restart`
Now you are only able to login using `ssh grader@35.174.190.27 -p 2200 -i .ssh/id_rsa`

9.__Install Apache__
`$sudo apt-get install apache2`

10.__Install mod_wsgi__
Run `$sudo apt-get install libapache2-mod-wsgi-py3`
Enable mod_wsgi with `$sudo a2enmod wsgi`
Start the web server with `$sudo service apache2 start`

11.__Clone the repository that contains Project 3 Catalog app__
Install git using: `$sudo apt-get install git`
`$cd /var/www`
`$sudo mkdir catalog`
Change owner of the newly created catalog folder `$sudo chown -R grader:grader catalog`
`$cd /catalog`
Clone your project from github `git clone https://github.com/pshegde123/BookStoreItemCatalog-App.git catalog`
Rename application.py to __init.py__.

12.__Create a catalog.wsgi file__
At path /var/www/catalog save following code in a file and save it as 'catalog.wsgi'
`import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'supersecretkey'`

13.__Install Flask and other dependancies __
Go to path /var/www/catalog.
Install the virtual environment `$sudo pip install virtualenv`
Create a new virtual environment with `$sudo virtualenv -p python3 py3env`
Activate the virutal environment `$source venv/bin/activate`
Change permissions `$sudo chmod -R 777 py3env`
Install Flask and other dependencies,
Install project dependencies `$sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils requests`

14. __Change client_secrets.json path__
Open __init.py__ and change client_secrets.json path to `/var/www/catalog/catalog/client_secrets.json`

15. __Configure and enable a new virtual host__
Run this: `$sudo nano /etc/apache2/sites-available/catalog.conf`
Paste this code:
`<VirtualHost *:80>
    ServerName 35.174.190.27
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
</VirtualHost>`
Enable the virtual host `$sudo a2ensite catalog`

14. __Install and configure PostgreSQL__
Install PostgreSQL with:
`$sudo apt-get install postgresql postgresql-contrib`
To ensure that remote connections to PostgreSQL are not allowed, I checked that the configuration file /etc/postgresql/9.5/main/pg_hba.conf only allowed connections from the local host addresses 127.0.0.1 for IPv4 and ::1 for IPv6.

PostgreSQL comes with a default username/password : postgres/postgres
Connect to default database 'psql' using command:
`$sudo -u postgres psql postgres`

Once you are connected to database, execute  following commands to create new user 'catalog' and a new database 'catalog':
` CREATE USER catalog WITH PASSWORD 'password';
ALTER USER catalog CREATEDB;
CREATE DATABASE catalog WITH OWNER catalog;
\c catalog
 GRANT ALL ON SCHEMA public TO catalog; 
\q `

Change create_engine code in your __init__.py, database_setup.py and data.py to: 
`engine = create_engine('postgresql://catalog:password@localhost/catalog')`

Execute following commands to create 'catalog' database schema and insert demo data in tables:
`python /var/www/catalog/catalog/database_setup.py`
`python /var/www/catalog/catalog/data.py`

Make sure no remote connections to the database are allowed. Check if the contents of this file sudo nano /etc/postgresql/9.3/main/pg_hba.conf looks like this:
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
Restart Apache
sudo service apache2 restart

Visit site at http://35.174.190.27

