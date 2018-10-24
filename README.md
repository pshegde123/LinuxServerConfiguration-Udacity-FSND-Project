# LinuxServerConfiguration
This is Udacity Full Stack Web Development Nanodegree course project. Purpose of the project is to configure a Linux server to host a web app securely.

Here are the details of the server (hosted on [Amazon LightSail](https://lightsail.aws.amazon.com/ls/webapp/home/instances)):
* Server IP address : 35.174.190.27
* SSH port : 2200
* Application URL : http://35.174.190.27.xip.io

## Configuration changes
1. __Set up PuTTY to connect server using SSH:__

   Follow the steps mentioned on [Amazon LightSail]( https://lightsail.aws.amazon.com/ls/docs/en/articles/lightsail-how-to-set-up-putty-to-connect-using-ssh ) and connect to the server IP: 35.174.190.27, port:22



2. __Create a new user named 'grader' and give it the permission to sudo:__
  * Run ` $sudo adduser grader ` to create a new user named _grader_
  * Create a new file _grader_ in the sudoers directory `/etc/sudoers.d` using command ` $sudo nano /etc/sudoers.d/grader `
  * Save following text in to the file ` grader ALL=(ALL) NOPASSWD:ALL `.



3. __Enable key based authentication:__
  * On your local machine generate a new SSH authentication key using command ` $ssh-keygen `
  * I have used passphrase as _udacity_.
  * Log in to linux server as ubuntu@35.174.190.27, change user to 'grader' using command ` $su - grader `
  * Execute following commands:
      1. `$mkdir /home/grader/.ssh`
      2. `$touch /home/grader/.ssh/authorized_keys`
      3. `$chown grader:grader /home/grader/.ssh`
      4. `$chmod 700 /home/grader/.ssh`
      5. Copy the public key generate in first step to `/home/grader/.ssh/authorized_keys`
      6. `$chown grader:grader /home/grader/.ssh/authorized_keys`
      7. `$chmod 644 /home/grader/.ssh/authorized_keys` 
  * From your local machine connect to server using command ` ssh grader@35.174.190.27 -i .ssh/id_rsa `
  * Enter passphrase _udacity_



4. __Update all currently installed packages on the server:__
* Download package lists with ` $sudo apt-get update `
* Fetch new versions of packages with ` $sudo apt-get upgrade `



5. __Change SSH port from 22 to 2200:__ 
* Open SSH config file using command ` $sudo nano /etc/ssh/sshd_config `
* Change the port from 22 to 2200
* Restart SSH service by executing command ` $sudo service ssh restart`
* Confirm change is working by command `$netstat -ntlp `, it should show post 2200 listening.  



6. __Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123):__
* By default, block all incoming connections on all ports:` $sudo ufw default deny incoming`
* Allow outgoing connection on all ports: `$sudo ufw default allow outgoing`
* Allow incoming connection for SSH on port 22 and 2200: `$sudo ufw allow ssh` and `$sudo ufw allow 2200/tcp`
* Allow incoming connections for HTTP on port 80: `$sudo ufw allow www`
* Allow incoming connection for NTP on port 123: `$sudo ufw allow ntp`
* To enable the firewall, use command `$sudo ufw enable`
* To check the status of the firewall, use command `$sudo ufw status`



7. __Configure the local timezone to UTC:__
* Run ` $sudo timedatectl set-timezone UTC `



8. __Disable SSH login for _root_ user:__
* Run ` $sudo nano /etc/ssh/sshd_config`
* Change ` PermitRootLogin without-password` line to `PermitRootLogin No`
* Restart ssh with `$sudo service ssh restart`
* Now you are only able to login using `ssh grader@35.174.190.27 -p 2200 -i .ssh/id_rsa`



9. __Install Apache:__
`$sudo apt-get install apache2`



10. __Install mod_wsgi:__
* Run `$sudo apt-get install libapache2-mod-wsgi-py3`
* Enable mod_wsgi with `$sudo a2enmod wsgi`
* Start the web server with `$sudo service apache2 start`


11. __Clone the repository that contains Catalog app:__
 * Execute following command to clone github project: 
```$sudo apt-get install git 
     $cd /var/www
     $sudo mkdir catalog
     $sudo chown -R grader:grader catalog
     $cd /var/catalog
     $git clone https://github.com/pshegde123/BookStoreItemCatalog-App.git catalog
```    
* Rename application.py to "__init.py__"

12. __Create a catalog.wsgi file:__
   At path `/var/www/catalog` save following code in a file named '_catalog.wsgi_'
   
  ```import sys
  import logging
  logging.basicConfig(stream=sys.stderr)
  sys.path.insert(0, "/var/www/catalog/")

  from catalog import app as application
  application.secret_key = 'supersecretkey'
  ```
  
13. __Install Flask and other dependancies:__
* Go to directory `/var/www/catalog`
* Install the virtual environment `$sudo pip install virtualenv`
* Create a new virtual environment with `$sudo virtualenv -p python3 py3env`
* Activate the virutal environment `$source py3env/bin/activate`
* Change permissions `$sudo chmod -R 777 py3env`
* Install project dependencies `$sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils requests`


14. __Change client_secrets.json path:__
Open __init.py__ and change client_secrets.json path to `/var/www/catalog/catalog/client_secrets.json`



15. __Configure and enable a new virtual host:__

* To configure new virtual host create new file `catalog.conf` using command `$sudo nano /etc/apache2/sites-available/catalog.conf`

* Save following code in `catalog.conf`:
```<VirtualHost *:80>
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
</VirtualHost>
```
* Enable the new virtual host by executing command `$sudo a2ensite catalog`


16. __Install and configure PostgreSQL:__

* Install PostgreSQL using command:
`$sudo apt-get install postgresql postgresql-contrib`

* Connect to default database 'psql' using command:
`$sudo -u postgres psql postgres`

* Once you are connected to database, execute  following commands to create new user 'catalog' and a new database 'catalog':
``` CREATE USER catalog WITH PASSWORD 'password';
ALTER USER catalog CREATEDB;
CREATE DATABASE catalog WITH OWNER catalog;
\c catalog
 GRANT ALL ON SCHEMA public TO catalog; 
\q 
```

* Change create_engine code in your __init__.py, database_setup.py and data.py to: 
`engine = create_engine('postgresql://catalog:password@localhost/catalog')`

* Execute following commands to create 'catalog' database schema and insert demo data in tables:
`python /var/www/catalog/catalog/database_setup.py`
`python /var/www/catalog/catalog/data.py`

* Make sure no remote connections to the database are allowed. Check if the contents of this file `$sudo nano /etc/postgresql/9.3/main/pg_hba.conf` looks like this:
  ```
  local   all             postgres                                peer
  local   all             all                                     peer
  host    all             all             127.0.0.1/32            md5
  host    all             all             ::1/128                 md5
  ```

* Restart Apache server using command: 
`$sudo service apache2 restart`

17. __Update the Google OAuth client secrets file:__

Update the `Authorized Javascript Origins` field at Developer Console->Credentials, add the IP address "http://35.174.190.27.xip.io"
Download updated JSON file and save it in `/var/www/catalog/catalog` folder.


18. __Update the Facebook OAuth client secrets file:__

In the Facebook developers website, on the Settings page, the website URL needs to read "http://35.174.190.27.xip.io". Update section `App Domains` to value `xip.io` and `35.174.190.27.xip.io`
Then add "https://example.com/?SuperSpecializerAuth=Facebook" to the "Valid OAuth redirect URIs" field. Then save these changes.


Visit site at http://35.174.190.27

