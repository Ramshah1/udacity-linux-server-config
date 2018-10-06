# Linux Server Config

Installation of packages and libraries on a Linux Server to host a web application. It includes installing updates, securing server, creating user login and setting up database.

~~EC2 url: http://ec2-18-194-53-53.eu-central-1.compute.amazonaws.com~~

~~IP address: http://18.194.53.53/~~

**Edit**: The above instance has been shut down. So the above urls are no more accessible.

SSH Port: 2200

Login with: `ssh -i udacityServer grader@18.194.53.53 -p 2200`

## Configuration:

### Update all currently installed packages

`apt-get update` - to update the package indexes

`apt-get upgrade` - to actually upgrade the installed packages

### Add User
Add user `grader` to the system with
```
sudo adduser grader
```

### Add to sudo group
Provide superuser access to the user `grader` with following command
```
sudo usermod -aG sudo grader
```

### Create SSH Key Pair locally
Generate a Public-Private key pair on the local machine using the following command
```
ssh-keygen
```
Then provide the filename say `udacityServer` for the keys.

### Set up SSH Keys for grader user
```
mkdir .ssh
touch .ssh/authorized_keys
nano .ssh/authorized_keys
```
Add the contents of *public* key file i.e. the one with .pub extension to the `authorized_keys` file.
The add the required security permissions with the following commands:
```
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
```
### Disable Root Login
Update the `/etc/ssh/sshd_config` file to disable root Login and password authentication.
Update the contents of the file from `PermitRootLogin without-password` to `PermitRootLogin no`.
Also, make sure that the following setting is un-commented.
```
PasswordAuthentication no
```
Restart the service so changes take effect.
```
sudo service sshd restart
```
Now only grader user wil be able to login.

### Change Timezone to UTC
View the timezone with the `date` command. Update it to UTC if different using
```
sudo timedatectl set-timezone UTC
```

### Change SSH Port from 22 to 2200
Go to the Server Console and update the `Port 22` to `Port 2200`.
Edit the file `/etc/ssh/sshd_config` and change the line `Port 22` to `Port 2200`
Restart the service so changes take effect.
```
sudo service sshd restart
```
We will now login to the server using
```
ssh -i udacityServer grader@18.194.53.53 -p 2200
```

### Configuring Uncomplicated Firewall (UFW)
By default, block all incoming connections on all ports
```
sudo ufw default deny incoming
```
Allow outgoing connection on all ports:
```
sudo ufw default allow outgoing
```
Allow incoming connection for SSH on port 2200:
```
sudo ufw allow 2200/tcp
```
Allow incoming connections for HTTP on port 80:
```
sudo ufw allow www
```
Allow incoming connection for NTP on port 123:
```
sudo ufw allow ntp
```
To check the rules that have been added before enabling the firewall use:
```
sudo ufw show added
```
To enable the firewall, use:
```
sudo ufw enable
```
To check the status of the firewall, use:
```
sudo ufw status
```

### Install Apache 
Install Apache and mod_wsgi using the following command
```
sudo apt-get install apache2
sudo apt-get install libapache2-mod-wsgi
```

### Install PostgresSQL
Install the Postgres with the following command
```
sudo apt-get install postgresql postgresql-contrib
```
### Install Other Dependencies
Now install Flask, SQLAlchemy, oAuth client to serve all dependencies.
```
sudo apt-get install python-psycopg2 python-flask
sudo apt-get install python-sqlalchemy python-pip
sudo pip install oauth2client
sudo pip install requests
sudo pip install httplib2
```

### Install Version Control Software
Git is used as the version control in this project. Install it using the following command
```
sudo apt-get install git
```

### Create the Flask App
* Create a folder `catalog` in `/var/www/` folder.
* `cd` into the `catalog` folder and clone the git repository for `Item Catalog Project`.
* Create a file `catalog.wsgi`.
* Add the following code to the newly created file
```
import sys
sys.path.insert(0, '/var/www/catalog')
from catalog import app as application
```

### Configure Apache to serve a Python mod_wsgi application:
Edit the `/etc/apache2/sites-enabled/000-default.conf` file. This file tells Apache how to respond to requests, where to find the files for a particular site etc. Update the file by adding the following code between the tags
```
DocumentRoot /var/www/catalog
WSGIDaemonProcess catalog threads=1
WSGIScriptAlias / /var/www/catalog/catalog.wsgi

<Directory /var/www/catalog>
        WSGIProcessGroup catalog
        WSGIApplicationGroup %{GLOBAL}
        Order deny,allow
        Allow from all
</Directory>
```
Restart the web server for changes to take effect
```
sudo apachectl restart
```
### Configure the Postgres
* Update the create_engine line in `databse_setup.py`, and `catalog.py` by adding the following line
```
engine = create_engine('postgresql://catalog:catalog-pw@localhost/catalog)
```
* Change to default user postgres:
```
sudo su - postgres
```
* Connect to the system with `psql` command.
* Create user catalog: 
```
CREATE USER catalog WITH PASSWORD 'catalog-pw';
```
* Allow the user to create database: 
```
ALTER USER catalog CREATEDB;
```
* Check lists of roles using `\du`
* Create database using:
```
CREATE DATABASE catalog WITH OWNER catalog;
```
* Connect to database using : `\c catalog`
* Revoke all the rights:
```
REVOKE ALL ON SCHEMA public FROM public;
```
* Grant the access to catalog: 
```
GRANT ALL ON SCHEMA public TO catalog;
```
* Create the database schea by running `python database_setup.py`.
* To Exit from Postgresql using `\q`.
* To restart postgresql:
```sudo service postgresql restart
```

### Setting up oAuth Login
* Go to [hcidata](http://www.hcidata.info/host2ip.cgi) and get the host name of public IP address (18.194.53.53) which in this case is http://ec2-18-194-53-53.eu-central-1.compute.amazonaws.com.
* Enable the virtual host: `sudo a2ensite catalog`
* Restart the apache server: `sudo service apache2 restart`
* Go to [Google Developer Console](https://console.developers.google.com)
* Edit the Credentials
* Add your hostname ([ec2-18-194-53-53.eu-central-1.compute.amazonaws.com](http://ec2-18-194-53-53.eu-central-1.compute.amazonaws.com)) and Public IP ([18.194.53.53](http://18.194.53.53)) to the Authorised JavaScript origins.
* Add hostname ([ec2-18-194-53-53.eu-central-1.compute.amazonaws.com/oauth2callback](http://ec2-18-194-53-53.eu-central-1.compute.amazonaws.com/oauth2callback)) to Authorised redirect URIs.
* Save and Download the updated `client_secret.json` file.
* Update the relative path to absolute path in all the files where `client_secret.json` is called, such that it reads as
```
CLIENT_ID = json.loads(open('/var/www/catalog/client_secret.json', 'r').read())['web']['client_id']
```

### Resources Used

1. [Basic Flask Tutorial](http://www.bogotobogo.com/python/Flask/Python_Flask_HelloWorld_App_with_Apache_WSGI_Ubuntu14.php)
2. [Flask with Apache Server Tutorial](https://www.datasciencebytes.com/bytes/2015/02/24/running-a-flask-app-on-aws-ec2/)
3. [Initial Server Setup](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04)
4. [Timezone conversion](https://askubuntu.com/questions/138423/how-do-i-change-my-timezone-to-utc-gmt/138442)
5. [SQLAlchemy Engine Configuration](http://docs.sqlalchemy.org/en/rel_1_0/core/engines.html#postgresql)
6. [Setup PostgresSQL on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)

