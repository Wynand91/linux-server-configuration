# PROJECT: Linux Server Configuration

Baseline Installation of a Linux server hosting the Catalog application created earlier in the Nanodegree.

## _Important information:_
   - Application URL: [http://ec2-18-195-96-160.eu-central-1.compute.amazonaws.com/](http://ec2-18-195-96-160.eu-central-1.compute.amazonaws.com/)
   - IP address: 18.195.96.160 (**Do not access site only by IP address! - OAuth will not work**)
   - SSH port: 2200 
   

## Set up server on Amazon Lightsail
   1. Navigate to [Amazon Lightsail](https://aws.amazon.com/lightsail/)
   2. click _Create instance_.
   3. Click Ubuntu instance image, OS Only and Ubuntu 16.04 LTS.
   4. Choose a instance plan (Lowest tier is free for one month).
   5. Give instance a hostname.
   6. Click _Create_ button to create the instance.


## SSH into the server
   - For now you can SSH into server with the button on your instance, but let's set up private key, because that will soon change.
   
   1. download the Default Private Key from the Account menu on your instance to your local machine.
   2. Move the downloaded file into the ~/.ssh directory and rename it to _amazon_pk.rsa_
   3. Change permissions of file `chmod 600 ~/.ssh/amazon_pk.rsa`
   - You can now SSH into server with `ssh -i ~/.ssh/amazon_pk.rsa ubuntu@<instance_public_ip_address>`
   
   
## Upgrade packages:
   
   ```
   sudo apt-get update
   sudo apt-get upgrade
   sudo apt-get dist-upgrade
   ```

## Change SSH port from 22 to 2200:
   
  `sudo nano /etc/ssh/sshd_config`
   
 Search for Port and change it from 22 to 2200
 
## Configure Uncomplicated Firewall (UFW):

   1. `sudo ufw default deny incoming`
   2. `sudo ufw default allow outgoing`
   3. `sudo ufw allow 2200/tcp`
   4. `sudo ufw deny 22`
   5. `sudo ufw allow www`
   6. `sudo ufw allow ntp`  
    (You can double check changes with: `sudo ufw show added`)
   
   7. `sudo ufw enable`
   8. On Amazon lighsail Instance, under Networking tab, update firewall to match changes.
   
   **Note:** From now on you have to SSH into server with `ssh -i ~/.ssh/amazon_pk.rsa -p 2200 ubuntu@<instance_public_ip_address>` 
   
## Configure timezone:

   `sudo dpkg-reconfigure tzdata` and follow commands to find your timezone.
   
## Create grader user and assign sudo privilege:

   1. `sudo adduser grader` and enter password.
   2. `sudo visudo`
   3. Add this to file: `grader ALL=(ALL:ALL) ALL` underneath root permissions.
   
## Create SSH key for grader:

   **On local machine:**
   
   1. Run `ssh-keygen` and enter a file (path including) to save the key to.
        - I used: ~/.ssh/grader_keypair
        - Two files will be created in the ~/.ssh directory: grader_keypair and grader_keypair.pub
   2. Run `cat ~/.ssh/grader_keypair.pub` and copy contents of file.
   
   **Log into server as grader:**
   
   1. In home directory run: `mkdir .ssh`
   2. `nano .ssh/authorized_keys` and paste contents of grader_keypair.pub that you copied.
   3. Set permissions: `chmod 700 .ssh` and `chmod 644 .ssh/authorized_keys`
   
   - **Make sure key based authentication is forced and Deacativate root remote login:**
        1. `sudo nano /etc/ssh/sshd_config` and make sure 'passwordAuthentication' is set to no
        2. `sudo su`
        3. `nano /etc/ssh/sshd_config` and change 'PermitRootLogin' line to 'PermitRootLogin no'
        4. `sudo service ssh restart`
   
   **NOTE:** From now on you must SSH into server with keypair:
   `ssh -i ~/.ssh/grader_keypair -p 2200 grader@<instance_public_ip_address>`
   
## Install Apache to serve a mod_wsgi application
   
   1. `sudo apt-get install apache2`
   2. `sudo apt-get install libapache2-mod-wsgi-py3 python-dev`
   3. `sudo a2enmod wsgi` to enable mod_wsgi.
   4. `sudo service apache2 restart`
   
##  Install and configure PostgreSQL

   1. `sudo apt-get install postgresql`
   2. Make sure remote connections are not allowed:
        - `sudo nano /etc/postgresql/9.5/main/pg_hba.conf`
        - Only allow connections from localhost 127.0.0.1(IPv4) and ::1(IPv6)
   3. `sudo su postgres` to change to postgres user
   4. `psql`
   5. `postgres=# CREATE DATABASE catalog;` to create database 'catalog'.
   6. `postgres=# CREATE USER catalog;` to create user 'catalog'.
   7. `postgres=# ALTER ROLE catalog WITH PASSWORD 'password';` to assign password to user 'catalog'.
   8. `postgres=# GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;` so user 'catalog' has authority for database 'catalog'.
   9. `postgres=# \q` to quit.
   10. `exit` to change back to grader.
   
## Clone catalog application from github:

   1. `sudo apt-get install git` to make sure git is installed.
   2. `cd /var/www/`
   3. `sudo mkdir catalog`
   4. `sudo chown -R grader:grader catalog`
   5. `cd catalog`
   6. `git clone <catalog-github-url> catalog`
   7. Make .git inaccessible: `sudo nano .htaccess -otsop` and add 'RedirectMatch 404 /\.git'
   8. Rename `application.py` to `__init__.py`: `mv application.py __init__.py`
   9. On `__init.py__` and `models.py`: change database engine to: `engine = create_engine('postgresql://catalog:password@localhost/catalog')`
   10. Run `python3 models.py` to create database schema.
   
## Configure wsgi file:

   1. `sudo nano /var/www/catalog/catalog.wsgi`
   2. Add the following and save file:
   ```
   #!/usr/bin/python3

   import sys
   import logging

   logging.basicConfig(stream=sys.stderr)
   sys.path.insert(0, "/var/www/catalog/")

   from catalog import app as application
   ```
   
## Install a virtual environment + dependencies:

   1. `sudo apt-get install python3-pip`
   2. `sudo apt-get install libpq-dev`
        - At some point here I got a "unsupported settings" error, which I fixed by running: `export LC_ALL=C`
   3. `python3 -m venv python3` to create virtual environment.
   4. `source python3/bin/activate` to activate virtual environment.
   5. Make sure you are in /var/www/catalog/catalog/ and run `pip3 install -r requirements.txt` to install requirements.
   
## Setup virtual host:
   1. `sudo nano /etc/apache2/sites-enabled/000-default.conf`
   2. Change file to this:
       ```
       <VirtualHost *:80>
            ServerName 18.195.96.160
            ServerAlias ec2-18-195-96-160.eu-central-1.compute.amazonaws.com
            ServerAdmin admin@18.195.96.160
            DocumentRoot /var/www/catalog
            WSGIDaemonProcess catalog user=grader group=grader
            WSGIProcessGroup catalog
            WSGIApplicationGroup %{GLOBAL}
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
   
   3. Add additional directories to search for Python modules.
        ```
        - sudo nano /etc/apache2/mods-enabled/wsgi.conf
        - look for this line: # WSGIPythonPath directory|directory-1:directory-2:...
        - Below the line add: WSGIPythonPath /var/www/catalog/python3/lib/python3.5/site-packages
        ```
   4. `sudo a2ensite 000-default` to enable virtual host.
   
## Update OAuth credentials:

   1. Go to [Google developer tools](https://console.developers.google.com)
   2. Change authorized javascript origins to: `http://ec2-18-195-96-160.eu-central-1.compute.amazonaws.com`
   3. Change authorized redirect URI's to: `http://ec2-18-195-96-160.eu-central-1.compute.amazonaws.com/oauth2redirect`
   4. On client_secret.json file by "javascript_origins" add `http://ec2-18-195-96-160.eu-central-1.compute.amazonaws.com`
   5. Change all uses of "client_secret.json" on `__init__.py` to full path: _'/var/www/catalog/catalog/client_secret.json'_
   
   
## Additional changes:
   - On `__init.py__` change:
        ```
            if __name__ == '__main__':
                app.debug = True
                app.run(host='0.0.0.0', port=8000)
        
        to:
            if __name__ == '__main__':
                app.run()
        
        ``` 
        
## Finishing up:

    - sudo service apache2 reload
    - sudo service apache2 restart
    
   App is now up and running at [http://ec2-18-195-96-160.eu-central-1.compute.amazonaws.com/](http://ec2-18-195-96-160.eu-central-1.compute.amazonaws.com/)

## Author:
   - Wynand Theron    
   
## Resources:
- [digitalocean - How To Deploy a Flask Application on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
- [medium - Deploying Flask Application to Amazon LightSail](https://medium.com/@zionoyemade/deploying-flask-application-to-amazon-lightsail-199e79bb256a)
- [medium - postgres configuration](https://medium.com/@dushan14/create-a-web-application-with-python-flask-postgresql-and-deploy-on-heroku-243d548335cc)
- [techonthenet - postgres](https://www.techonthenet.com/postgresql/grant_revoke.php)
- [postgresql docs](https://www.postgresql.org/docs/)
- [digitalocean - How To Secure PostgreSQL on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)
- [askubuntu - changing timezone](https://askubuntu.com/questions/138423/how-do-i-change-my-timezone-to-utc-gmt/138442)
- [Udacity Fullstack Nanodegree - Deploying to Linux Servers](https://classroom.udacity.com/)
- [Hiding git repo](https://davidegan.me/hide-git-repos-on-public-sites/)
- [mod_wsgi(Apache)](http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/)
- [digitalocean - Apache virtual hosts](https://www.digitalocean.com/community/tutorials/how-to-set-up-apache-virtual-hosts-on-ubuntu-14-04-lts)
- [WSGIPythonPath](https://modwsgi.readthedocs.io/en/develop/configuration-directives/WSGIPythonPath.html)
- [rrjoson - github](https://github.com/rrjoson/udacity-linux-server-configuration)
- [granting sudoers permission](https://www.garron.me/en/linux/visudo-command-sudoers-file-sudo-default-editor.html)

## License:

MIT License

Copyright (c) 2019 Wynand Theron

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
        

   
   
      
   
