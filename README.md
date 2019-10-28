# Linux Server Configuration

## Project Overview

this is the final project of udacity Full stack developer nanodegree

we will take a Linux server and prepare it to host my web applications using Amazon Lighsail. after that, we will secure the server from a different type of attack, install and configure a database server, and finally deploy the catalog application onto it.

to do that, we will follow these steps :

1- Login into Amazon Lightsail using an Amazon Web Services account and create a new instance, after that connect to it using ssh

2- Update Available Package Lists using those commands:

* `sudo apt-get update` (run this command as root user adding sudo)
* Upgrading Installed Packages sudo apt-get dist-upgrade `sudo apt-get dist-upgrade`
* shut down the server to apply changes : sudo shutdown -r now `sudo shutdown -r now`

3- Change the SSH port from 22 to 2200 following these steps :

* edit the sshd_config file using `sudo nano /etc/ssh/sshd_config` after locate the port number and change it to 2200

* save the changes and restart the ssh service using `sudo service ssh restart`

* if we want to connect to the server another time via the terminal we use the following command : `ssh -i ~/.ssh/lightsail_key.rsa -p 2200 ubuntu@15.188.77.10` , (15.188.77.10 is the public IP).

4- Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123), we follow these steps :

* block all incoming request(or everything coming in) : `sudo ufw default deny incoming`
* establish a default rule for the outgoing connection : `sudo ufw default allow outgoing`
* allow ssh and tcp connection through port 2200: `sudo ufw allow ssh` /  `sudo ufw allow 2200/tcp`
* support basic http server : `sudo ufw allow www`
* support NTP connection : `sudo ufw allow ntp`
* enable firewall :  `sudo ufw enable`

5- create the grader user and give it the sudo access :

* to create a new user we use the following command: `sudo adduser grader`
* install finger to output all information about all of user currently logged into the system `sudo apt-get install finger`
* give the grader user the sudo access, to do that, create the grader file under sudoers.d directory `sudo nano /etc/sudoers.d/grader` and add this line :  `grader ALL=(ALL:ALL) ALL` after that save the file.

6- Configure the key-based authentication for grader user :

* generate key-pair in your local machine using ssh-keygen `ssh-keygen -f ~/.ssh/grader_key.rsa`
* create a .ssh directory in the grader directory : `sudo mkdir /home/grader/.ssh`
* create a file authorized_keys in the .ssh directory : `sudo touch /home/grader/.ssh/authorized_keys`
* Copy the content of the grader_key.pub file from your local machine to the /home/grader/.ssh/authorized_keys file, finally setup some file permission on the authorized_keys and ssh directory :
    1. `sudo chmod 700 /home/grader/.ssh`
    2. `sudo chmod 644 /home/grader/.ssh/authorized_keys`
    3. change the owner from ubuntu to grader `sudo chown -R grader:grader /home/grader/.ssh`

* finally, connect to the server using this command: `ssh -i ~/.ssh/grader_key.rsa -p 2200 grader@15.188.77.10`
* to enforce the Key-based SSH authentication, edit sshd_config and find the line PasswordAuthentication and edit it to no :
    1. `sudo nano /etc/ssh/sshd_config`
    2. `sudo service ssh restart`

7- Disable remote login of the root user :

* edit sshd_config and find the line PermitRootLogin and edit it to no :
    1. `sudo nano /etc/ssh/sshd_config`
    2. `sudo service ssh restart`

8- install the required software to deploy and start the catalog app :

* install git : `sudo apt-get install git`
* install PostgreSQL : `sudo apt-get install postgresql`
* install Apache : `sudo apt-get install apache2`
* Install the virtual environment: `sudo apt-get install python-virtualenv`
* install the Python 3 mod_wsgi package :  `sudo apt-get install libapache2-mod-wsgi-py3`
* install Flask and the project's dependencies :
  1. install pip, the tool for installing Python packages: `sudo apt-get install python-pip`
  2. install Flask: `pip install Flask`
  3. Install all the other project's dependencies: $ `pip install httplib2 request oauthlib sqlalchemy psycopg2-binary flask_login`.

9- configure PostgreSQL:

* Switch to the postgres user: `sudo su - postgres` and run `psql`
* create a new PostgreSQL role to allow us to connect our ubuntu account to our database : `createuser --interactive`
  
* create a database with the same name as our user account : `createdb catalog`
* change the owner of the database : `ALTER DATABASE catalog OWNER TO catalog;`
* set a new password for the new user account before proceeding: `ALTER USER catalog WITH PASSWORD 'password';`


10- clone the Item Catalog project and populate the database with some initial value :

* create /var/www/catalog/ directory with the command : `sudo mkdir /var/www/catalog/`
* go to the catalog directory and clone the github project in it : `sudo git clone https://github.com/baza1/udacity-catalog-app.git catalog`
* go to the catalog project and create the tables of the database with this command : `python database_setup.py`
* To populate the database run `python init_db_rows.py`

11- prepare the deployment of the application

* Change to the `/var/www/catalog/catalog/` directory and Create the virtual environment: `sudo virtualenv -p python3 virenv`

* go to : `/etc/apache2/mods-enabled/` and add in the `wsgi.conf` file the following line to use Python 3, add it just below this line `#WSGIPythonPath directory|directory-1:directory-2:...` : `WSGIPythonPath /var/www/catalog/catalog/venv3/lib/python3.5/site-packages`
  
* to Set up the Flask application, create the catalog.wsgi in the /var/www/catalog/ directory and add these lines :

```
activate_this = '/var/www/catalog/catalog/virenv/bin/activate_this.py'
with open(activate_this) as file_:
    exec(file_.read(), dict(__file__=activate_this))

#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/catalog/")
sys.path.insert(1, "/var/www/cvenv3/atalog/")

from project import app as application
application.secret_key = "..."
```

* to configure the virtualhost, create the catalog.conf file in the /etc/apache2/sites-available/ directory and add the following lines : 

```
<VirtualHost *:80>
    ServerName 15.188.77.10
    ServerAlias ec2-15-188-77-10.eu-west-3a.compute.amazonaws.com
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

* enable the virtual host with the following command: `sudo a2ensite catalog` and reload apache `sudo service apache2 reload`
* start the application at : http://15.188.77.10/

## reference that i used during this project

* https://www.cyberciti.biz/faq/howto-change-ssh-port-on-linux-or-unix-server/
* https://github.com/iliketomatoes/linux_server_configuration
* https://askubuntu.com/questions/27559/how-do-i-disable-remote-ssh-login-as-root-from-a-server
* https://wixelhq.com/blog/how-to-install-postgresql-on-ubuntu-remote-access
* https://www.digitalocean.com/community/tutorials/how-to-serve-django-applications-with-apache-and-mod_wsgi-on-ubuntu-16-04
* https://modwsgi.readthedocs.io/en/develop/user-guides/quick-configuration-guide.html
* https://github.com/boisalai/udacity-linux-server-configuration