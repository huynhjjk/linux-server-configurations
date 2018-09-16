# Project 6: Linux Server Configuration
A tutorial on how to securely configure a linux server and host a web application. This is the sixth and final project assignment from [Udacity's Full Stack Web Developer Nanodegree](https://www.udacity.com/nanodegree). 

- SSH Port: 2200
- IP Address: 34.221.113.189
- Server URL: http://ec2-34-221-113-189.us-west-2.compute.amazonaws.com/

## Summary of Software Installed
- apache2
- flask
- git
- httplib2
- libapache2-mod-wsgi
- oauth2client
- postgresql
- python-dev
- python-pip
- python-psycopg2
- request 
- sqlalchemy
- virtualenv

## Log into the server as the user grader using the submitted key
- Download the grader.pem provided by owner
```
mv grader.pem ~/.ssh/grader.pem
ssh grader@34.221.113.189 -p 2200 -i ~/.ssh/grader.pem
```

## Update All System Packages to Most Recent Versions
    sudo apt-get update
    sudo apt-get upgrade
    sudo apt-get dist-upgrade
    sudo shutdown -r now

## Disable Remote Login of The Root User
    sudo nano /etc/ssh/sshd_config.config
- Set ```permitRootLogin``` to ```no```

## Enforce key-based SSH authentication
    sudo nano /etc/ssh/sshd_config.config
- Set ```passwordAuthentication``` to ```no```

## Set SSH to host on non-default port
    sudo nano /etc/ssh/sshd_config.config
- Set ```Port``` to ```2200```

## Configure Firewall to Only Allow Connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
- Enter the following on your virtual machine
    ```
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow 2200/tcp
    sudo ufw allow www
    sudo ufw allow 123/udp
    sudo ufw deny 22
    sudo ufw enable
    sudo ufw status
    ```
- Visit https://lightsail.aws.amazon.com/ls/webapp/home/instances
- Click Networking tab and add the following
    ```
    HTTP TCP 80
    CUSTOM UDP 123
    CUSTOM TCP 2200
    ```

## Configure Timezone to UTC
    sudo dpkg-reconfigure tzdata

## Create User Grader with key-based authentication
- Initially create the user ```grader```
    ```
    sudo adduser grader
    ```
- Give sudo permissions to the user ```grader```
    ```
    sudo nano /etc/sudoers.d/grader
    ```
- Enter the following rules
    ```
    grader ALL=(ALL:ALL) ALL
    ```
- Login as grader
    ```
    sudo su - grader
    ```
- Generate a private & public key on local machine
    ```
    ssh-keygen ~/.ssh
    ```
- Rename the newly created private key file to grader.pem
    ```
    mv grader grader.pem
    ```
- Copy the contents in grader.pub from local machine
    ```
    cat ~./ssh/grader.pub
    ```
- Paste the contents from grader.pub onto authorized_keys on your virtual machine
    ```
    sudo nano ~/.ssh/authorized_keys
    ```
- Give access rights to grader
    ```
    chown grader:grader /home/grader/.ssh/
    ```
- SSH as user grader
    ```
    ssh grader@34.221.113.189 -p 2200 -i ~/.ssh/grader.pem
    ```

## Configure the database server to serve data in PostgreSQL
- Install PostgreSQL
    ```
    sudo apt-get install postgresql
    ```
- Login as postgres
    ```
    sudo su - postgres
    ```
- Connect to the database using ```psql```
    ```
    psql
    ```
- Create a user ```catalog``` with password ```catalog```
    ```
    CREATE USER catalog WITH PASSWORD 'catalog';
    ```
- Create a database for catalog with owner as catalog
    ```
    CREATE DATABASE catalog WITH OWNER catalog;
    ```
- Connect to the newly created catalog database
    ```
    \c catalog
    ```
- Revoke all public rights
    ```
    REVOKE ALL ON SCHEMA public FROM public;
    ```
- Grant permission to only catalog user
    ```
    GRANT ALL ON SCHEMA public TO catalog;
    ```

## Create Catalog Application
- Install git
    ```
    sudo apt-get install git
    ```
- Access www directory
    ```
    cd /var/www
    ```
- Create a new directory called catalog
    ```
    sudo mkdir catalog
    ```
- Change owner for catalog directory
    ```
    sudo chown -R grader:grader catalog
    ```
- Access the first catalog directory
    ```
    cd catalog
    ```
- Clone the catalog repository with the directory named ```catalog```
     ```
     git clone https://github.com/huynhjjk/udacity-catalog catalog
     ```
- From ```catalog``` directory, ```cd``` into the second catalog directory containing files from branch
    ```
    cd catalog
    ```
- Rename python file
    ```
    mv application.py __init__.py
    ```
- Change all relative paths for ```client_secrets.json```
    ```
    sudo nano __init__.py
    ```
- ```client_secrets.json``` to ```/var/www/catalog/catalog/client_secrets.json```
- Change database connections on ```__init__.py```, ```database_setup.py```, ```load_database.py```
    ```
    sudo nano __init__.py
    postgresql://catalog:catalog@localhost/catalog
    sudo nano database_setup.py
    postgresql://catalog:catalog@localhost/catalog
    sudo nano load_database.py
    postgresql://catalog:catalog@localhost/catalog
    ```

## Configure the web server to serve the Item Catalog application as a WSGI app
- Install apache2
    ```
    sudo apt-get install apache2
    ```
- Install libapache2-mod-wsgi
    ```
    sudo apt-get install libapache2-mod-wsgi
    ```
- Enable wsgi
    ```
    sudo a2enmod wsgi
    ```
- Start apache2
    ```
    sudo service apache2 start
    ```
- ```cd``` into to the catalog folder
    ```
    cd /var/www/catalog
    ```
- Create a catalog.wsgi file and enter
    ```
    sudo nano catalog.wsgi
    import sys
    import logging

    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0, "/var/www/catalog/")

    from catalog import app as application
    application.secret_key = "<SUPER SECRET KEY>"
    ```
- ```sudo nano /etc/hosts``` and add ```ec2-34-221-113-189.us-west-2.compute.amazonaws.com```
- Create a virtual host config file ```sudo nano /etc/apache2/sites-available/catalog.conf``` and enter
    ```
    <VirtualHost *:80>
        ServerName 34.221.113.189
        ServerAlias ec2-34-221-113-189.us-west-2.compute.amazonaws.com
        ServerAdmin huynhjjk@gmail.com
        WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
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
- Enable virtual host: ```sudo a2ensite catalog```

## Configure Virtual Environment 
- ```cd``` into the catalog folder
   ```
    cd /var/www/catalog
   ```
- Install pip
    ```
    sudo apt-get install python-pip
    ```
- Install virtualenv
    ```
    sudo pip install virtualenv
    ```
- Create a virtual environment called ```venv```
    ```
    sudo virtualenv venv
    ```
- Activate the newly created virtual environment
    ```
    source venv/bin/activate
    ```
- Install the following dependencies:
    ```
    sudo pip install flask
    sudo pip install httplib2
    sudo pip install oauth2client
    sudo pip install psycopg2
    sudo pip install requests
    sudo pip install sqlalchemy
    ```

# Authenticate login with Google
- Visit https://console.cloud.google.com/apis/credentials of your application
- Add the following under ```Authorized Javascript Origins```
    ```
    http://ec2-34-221-113-189.us-west-2.compute.amazonaws.com
    http://34.221.113.189
    ```
- Add the following under ```Authorized redirect URIs```
    ```
    http://ec2-34-221-113-189.us-west-2.compute.amazonaws.com/login
    http://ec2-34-221-113-189.us-west-2.compute.amazonaws.com/gconnect
    ```

# Third-Party Resources
- https://github.com/boisalai/udacity-linux-server-configuration

## Authors
- Johnson Huynh
