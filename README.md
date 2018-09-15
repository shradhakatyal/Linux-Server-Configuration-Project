# Linux Server Configuration Project

## Application Information

- IP Address: http://13.233.43.112/
- URL: https://ec2-13-233-43-112.ap-south-1.compute.amazonaws.com/
- SSH Port: 2200

## Steps to set up server and deploy project

1. Update all packages
- Enter the following commands to update all the packages.
```
sudo apt-get update
sudo apt-get upgrade
```

2. To change SSH port from 22(default) to 2200
- Go to lightsail instances dashboard.
- Click on manage and then go to Networking tab.
- Under the Firewall heading click on edit rules and add a new rule to allow port 2200.
- cd into /etc/ssh and edit the sshd_config file using the command sudo nano sshd_config
- change port 22 to 2200

3. Configuring the Uncomplicated Firewall to allow only port 80, 123 and 2200
- sudo ufw default deny incoming
- sudo ufw default allow outgoing
- sudo ufw allow 2200
- sudo ufw allow 80
- sudo ufw allow 123
- sudo ufw enable

4. Adding new user Grader, giving grader sudo access and enabling key based authentication
- To add a new user, enter the command: adduser grader
- Switch to local terminal, type ssh-keygen and enter path for new keys. Two new keys will be created, a private and a public key. We will save the public key on our server.
- Now save the public key onto grader account by creating new folder .ssh and new file in it authorized_keys.
- Paste public key in authorized_keys.
- Chmod 700 for .ssh and chmod 644 for authorized_keys.
- Change owner of the folder to grader: sudo chown -R grader:grader /home/grader/.ssh
- To give sudo access to grader, add a new file called grader in sudoers.d;
- Edit the grader file: sudo nano /etc/sudoers.d/grader and enter the line: grader ALL=(ALL) NOPASSWD:ALL
- Restart ssh.
- The grader can then log in via ssh: ssh -i UdacityGraderKey.pem grader@13.233.43.112 -p 2200

5. Configuring local timezone to UTC
- Enter sudo dpkg-reconfigure tzdata in terminal
- Select none of the above in the first list.
- Select UTC in the second list.

6. Preparing to deploy project
Enter the following commands to install the required packages:
- sudo apt-get install apache2
- sudo apt-get install git
- sudo apt-get install libapache2-mod-wsgi
- You then need to configure Apache to handle requests using the WSGI module. You’ll do this by editing the /etc/apache2/sites-enabled/000-default.conf file.
- Check if no remote connections are allowed by entering the command: sudo vim /etc/postgresql/9.5/main/pg_hba.conf
- Enable mod_wsgi: $ sudo a2enmod wsgi.
- Start server: sudo service apache2 start

7. Installing Postgresql and setting up user catalog
- sudo apt-get install postgresql
- sudo su - postgres
- Enter psql in terminal and then run the following commands.
- postgres=# create database catalog;
- postgres=# create user catalog;
- postgres=# alter role catalog with password 'udacity';
- postgres=# grant all privileges on database catalog to catalog;
- \q - to quit psql
- exit postgres user by typing exit.

8. Deploying the project on server
- Go to /var/www
- Clone project here: git clone https://github.com/shradhakatyal/Item-Catalog.git
- cd into Item-Catalog
- Install virtualenv using the command: pip install virtualenv
- Create a new virtual environment: virtualenv venv
- Activate the virtual environment: source venv/bin/activate
- cd into catalog folder inside of Item-Catalog
- To install all the dependencies of the project: pip install -r requirements.txt
- Change app.py to __init__.py by typing sudo mv app.py init.py
- Change engine = create_engine('sqlite:///itemcatalog.db') to engine = create_engine('postgresql://catalog:udacity@localhost/catalog')
in __init__.py, populate_db.py and database_setup.py
- Chane the path of client_secrets.json in __init__.py to 'var/www/Item-Catalog/catalog/client_secrets.json'
-Change owner to grader: sudo chown -R grader:grader Item-Catalog

9. Creating wsgi file
- cd into Item-Catalog
- Create new file Item-Catalog.wsgi and enter the following code in it
```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/Item-Catalog/")

from catalog import app as application
application.secret_key = 'super_secret_key'
```

10. Creating Virtual Host
- sudo nano /etc/apache2/sites-available/Item-Catalog.conf to create config file and add the following lines in it.
```
<VirtualHost *:80>
    ServerName 13.233.43.112
    ServerAlias ec2-13-233-43-112.ap-south-1.compute.amazonaws.com
    ServerAdmin admin@13.233.43.112
    WSGIDaemonProcess catalog python-path=/var/www/Item-Catalog:/var/www/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/Item-Catalog/Item-Catalog.wsgi
    <Directory /var/www/Item-Catalog/catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/Item-catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel debug
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
- Enable the virtual host: sudo a2ensite catalog
- Restart apache2: sudo service apache2 restart

## External Links used to complete the project
- https://askubuntu.com/questions/138423/how-do-i-change-my-timezone-to-utc-gmt
- https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
- https://stackoverflow.com/questions/14604699/how-to-activate-virtualenv