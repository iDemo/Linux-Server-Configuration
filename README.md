# Udacity - Linux Server Configuration Project

in this project I will do an installation of Ubuntu Linux on a virtual machine to host a Flask web application, this includes the installation of updates, securing the system from a number of attack vectors and installing/configuring web and database servers.


## Website Info

IP address: 18.184.144.70
Accessible SSH port: 2200
Application URL: [http://ec2-18-184-144-70.eu-central-1.compute.amazonaws.com/](http://ec2-18-184-144-70.eu-central-1.compute.amazonaws.com/)



## Step by step walkthrough
### 1 - Create a new user named *grader* and grant this user sudo permissions.

1. Log into the remote VM as *root* user through ssh: `$ ssh root@18.184.144.70`.
2. Add a new user called *grader*: `$ sudo adduser grader`.
3. Create a new file under the suoders directory: `$ sudo nano /etc/sudoers.d/grader`. Fill that newly created file with the following line of text: "grader ALL=(ALL:ALL) ALL", then save it.


### 2 - Update and upgrade all currently installed packages
1. `$ sudo apt-get update`.
2. `$ sudo apt-get upgrade`.
3. Install *finger*, a utility software to check users' status: `$ apt-get install finger`.


### 3 - Configure the local timezone to UTC
1. Open time configuration dialog and set it to UTC with: `$ sudo dpkg-reconfigure tzdata`.
2. Install *ntp daemon ntpd* for a better synchronization of the server's time over the network connection: `$ sudo apt-get install ntp`.


### 4 - Configure the key-based authentication for *grader* user

1. Generate an encryption key **on your local machine** with: `$ ssh-keygen`
2. Enter the path and name for your key: `~/.ssh/ubuntu`
3. Log into the remote VM as *root* user through ssh and create the following file: `$ touch /home/grader/.ssh/authorized_keys`.
4. Copy the content of the *ubuntu.pub* file from your local machine to the */home/grader/.ssh/authorized_keys* file you just created on the remote VM. Then change some permissions:
	1. `$ sudo chmod 700 /home/grader/.ssh`.
	2. `$ sudo chmod 644 /home/grader/.ssh/authorized_keys`.
	3. Finally change the owner from *root* to *grader*: `$ sudo chown -R grader:grader /home/grader/.ssh`.
5. Now you are able to log into the remote VM through ssh with the following command: `$ ssh -i ~/.ssh/udacity_key.rsa grader@18.184.144.70`.


### 5 - Enforce key-based authentication
1. `$ sudo nano /etc/ssh/sshd_config`.
2. Find the **PasswordAuthentication** line and edit it to **no**.
3. `$ sudo service ssh restart`.

### 6 - Change the SSH port from 22 to 2200
1. `$ sudo nano /etc/ssh/sshd_config`. Find the *Port* line and edit it to *2200*.
2. `$ sudo service ssh restart`.
3. Now you are able to log into the remote VM through ssh with the following command: `$ ssh -i ~/.ssh/udacity_key.rsa -p 2200 grader@18.184.144.70`.


### 7 - Disable ssh login for *root* user
1. `$ sudo nano /etc/ssh/sshd_config`. Find the *PermitRootLogin* line and edit it to *no*.
2. `$ sudo service ssh restart`.


### 8 - Configure the Uncomplicated Firewall (UFW)

Project requirements need the server to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

1. `$ sudo ufw allow 2200/tcp`.
2. `$ sudo ufw allow 80/tcp`.
3. `$ sudo ufw allow 123/udp`.
4. `$ sudo ufw enable`.


### 9 - Install Apache, mod_wsgi

1. `$ sudo apt-get install apache2`.
2. Mod_wsgi is an Apache HTTP server mod that enables Apache to serve Flask applications. Install *mod_wsgi* with the following command: `$ sudo apt-get install libapache2-mod-wsgi python-dev`.
3. Enable *mod_wsgi*: `$ sudo a2enmod wsgi`.
3. `$ sudo service apache2 start`.

### 10 - Install Git

1. `$ sudo apt-get install git`.
2. Configure your username: `$ git config --global user.name <username>`.
3. Configure your email: `$ git config --global user.email <email>`.


### 13 - Clone the Catalog app from Github

1. `$ cd /var/www`. Then: `$ sudo mkdir catalog`.
2. Change owner for the *catalog* folder: `$ sudo chown -R grader:grader catalog`.
3. Move inside that newly created folder: `$ cd /catalog` and clone the catalog repository from Github: `$ git clone https://github.com/iDemo/catalogItems.git catalog`.
4. Make a *catalog.wsgi* file to serve the application over the *mod_wsgi*. That file should look like this:

```python
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")

from catalog import app as application
application.secret_key = 'Add your secret key'
```

5. Make the GitHub repository inaccessible:
Create and open *.htaccess* file:
`$ cd /var/www/catalog/` and `$ sudo vim .htaccess`
Paste in the following:
`RedirectMatch 404 /\.git`


### 14 - Install virtual environment, Flask and the project's dependencies

1. Install *pip*, the tool for installing Python packages: `$ sudo apt-get install python-pip`.
2. If *virtualenv* is not installed, use *pip* to install it using the following command: `$ sudo pip install virtualenv`.
3. Move to the *catalog* folder: `$ cd /var/www/catalog`. Then create a new virtual environment with the following command: `$ sudo virtualenv venv`.
4. Activate the virtual environment: `$ source venv/bin/activate`.
5. Change permissions to the virtual environment folder: `$ sudo chmod -R 777 venv`.
6. Install Flask: `$ pip install Flask`.
7. Install all the other project's dependencies: `$ pip install bleach httplib2 request oauth2client sqlalchemy python-psycopg2`. 



### 15 - Configure and enable a new virtual host

1. Create a virtual host conifg file: `$ sudo nano /etc/apache2/sites-available/catalog.conf`.
2. Paste in the following lines of code:
```
 <VirtualHost *:80>
      ServerName 18.184.144.70
      ServerAdmin admin@18.184.144.70
      ServerAlias ec2-18-184-144-70.eu-central-1.compute.amazonaws.com
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

3. Enable the new virtual host: `$ sudo a2ensite catalog`.



### 16 - Install and configure PostgreSQL

1. Install some necessary Python packages for working with PostgreSQL: `$ sudo apt-get install libpq-dev python-dev`.
2. Install PostgreSQL: `$ sudo apt-get install postgresql postgresql-contrib`.
3. Postgres is automatically creating a new user during its installation, whose name is 'postgres'. That is a tusted user who can access the database software. So let's change the user with: `$ sudo su - postgres`, then connect to the database system with `$ psql`.
4. Create a new user called 'catalog' with his password: `# CREATE USER catalog WITH PASSWORD 'sillypassword';`.
5. Give *catalog* user the CREATEDB capability: `# ALTER USER catalog CREATEDB;`.
6. Create the 'catalog' database owned by *catalog* user: `# CREATE DATABASE catalog WITH OWNER catalog;`.
7. Connect to the database: `# \c catalog`.
8. Revoke all rights: `# REVOKE ALL ON SCHEMA public FROM public;`.
9. Lock down the permissions to only let *catalog* role create tables: `# GRANT ALL ON SCHEMA public TO catalog;`.
10. Log out from PostgreSQL: `# \q`. Then return to the *grader* user: `$ exit`.
11. Inside the Flask application, the database connection is now performed with: 
```python
engine = create_engine('postgresql://catalog:catalogpass@localhost/catalog')
```
12. Setup the database with: `$ python /var/www/catalog/catalog/setup_database.py`.




### 17 - Update OAuth authorized JavaScript origins

1. To let users correctly log-in change the authorized URI to [http://ec2-18-184-144-70.eu-central-1.compute.amazonaws.com/](http://ec2-18-184-144-70.eu-central-1.compute.amazonaws.com/) on both Google and Facebook developer dashboards.

### 18 - Restart Apache to launch the app
1. `$ sudo service apache2 restart`.

### Resources:
##### Special thanks to:
- [*stueken*](https://github.com/stueken) who wrote a really helpful README in his [repository](https://github.com/stueken/FSND-P5_Linux-Server-Configuration).
- [*iliketomatoes*](https://github.com/stueken) for the great walkthrough in his project README in [repository](https://github.com/iliketomatoes/linux_server_configuration.git).
