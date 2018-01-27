# Linux Server Configuration project

We are deploying a flask application on to AWS Light Sail.  This document will give you steps from creating your instance to your application.  At the time when this document is written, ubuntu 16.06 LTS is the standard. Here is the final website link: [Item Catalog App](http://35.170.69.46/)

IP address: 35.170.69.46
Port: 2200


Please view this [github](https://github.com/rambo255/item_catalog.git) to use the application

## Creating your LightSail Instance for ubuntu

1. Login into [AWS Light Sail](https://lightsail.aws.amazon.com) using your provided email and password to amazon

2. Once you are login into the site, there is a button called create Instance. Click that button.

3. You'll be given two options: App+OS, OS only.

4. Choose OS only and ubuntu

5. Choose the instance plan and then click the button.

You can name your instance or just leave the default name provided by AWS .  They give an option to change the instance name.

Once we have created our ubuntu image, now we need to start configuring the server to deploy our flask application.

## Accessing the instance

Before setting up the server, we need to download the lightsail pem file from the LightSail website.  In order to get the file, you need to go the accounts page.
Once you are in the accounts page, there is an option to download the default private key.

To see if you can login into instance. Two things need to be done:

1. You need to change permission of the downloaded lightsail private key.
`chmod 400 LightsailDefaultPrivateKey.pem` or `chmod 600 LightsailDefaultPrivateKey.pem`

2. To login:
`ssh user@PublicIP -i LightsailDefaultPrivateKey`

## Updating the server
Since you are running ubuntu, its only two commands:

```
sudo apt-get update
sudo apt-get upgrade
```

## Creating the New user with sudo permissions
You are login into the instance. Lets create a new user called grader:
```
sudo adduser grader
```
It will prompt for a password. Add any password to that field.
After the user gets created, run this command to give the grader sudo priviledges:
```
sudo visudo
```
Once your in the visudo file, go to the section user privilege specification
```
root    ALL=(ALL:ALL) ALL
```
You've noticed the root user in the file. You need add the grader user to same section.  It should look like this:
```
grader  ALL=(ALL:ALL) ALL
```
If you want to login into the account, check to see if works: `sudo login grader`.
We've created our new user, lets configure our ssh to non-default port

## Configuring SSH to a non-default port

SSH default port is 22. We want to configure it to non default port.  Since we are using LightSail, we need to open the non default port through the web interface. If you don't do this step, you'll be locked out of the instance. Go to the networking tab on the management console. Then, the firewall option

Go into your sshd config file: `sudo nano /etc/ssh/sshd_config`
Change the following options. # means commented out

```
# What ports, IPs and protocols we listen for
# Port 22
Port 2200

# Authentication:
LoginGraceTime 120
#PermitRootLogin prohibit-password
PermitRootLogin no
StrictModes yes

# Change to no to disable tunnelled clear text passwords
PasswordAuthentication No
```
Once these changes are made, restart ssh: `sudo service restart ssh`
Any user can login into the machine using a specify port: `sudo user@PublicIP -p 2200`
 
## Set ssh login using keys
1. generate keys on local machine using`ssh-keygen` ; then save the private key in `~/.ssh` on local machine
2. deploy public key on developement enviroment

	On you virtual machine:
	```
	$ su - grader
	$ mkdir .ssh
	$ touch .ssh/authorized_keys
	$ vim .ssh/authorized_keys
	```
	Copy the public key generated on your local machine to this file and save
	```
	$ chmod 700 .ssh
	$ chmod 644 .ssh/authorized_keys
	```
	
3. reload SSH using `service ssh restart`
4. now you can use ssh to login with the new user you created

	`ssh -i [privateKeyFilename] grader@35.170.69.46` 

## Configuring Firewall rules using UFW
We need to configure firewall rules using UFW. Check to see if ufw is active: `sudo ufw status`. If not active, lets add some rules
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp or sudo ufw allow www (either one of these commands will work)
sudo ufw allow 123/udp
```
Now, you enable the rules: `sudo ufw enable` and re check the status to see what rules are activity

## Configure timezone for server
```
sudo dpkg-reconfigure tzdata
```
Choose none of the above and choose UTC.  The server by default is on UTC.

## Install Apache, Git, and flask

### Apache
We will be installing apache on our server. To do that:

```
sudo apt-get install apache2
```
If apache was setup correctly, a welcome page will come up when you use the PublicIP. We are going to install mod wsgi, python setup tools, and python-dev
```
sudo apt-get install libapache2-mod-wsgi python-dev
```
We need to enable mod wsgi if it isn't enabled: `sudo a2enmod wsgi`

Let's setup wsgi file and sites-available conf file for our application.
Create the WSGI file in `/var/www/FlaskApp/`
```
#!/usr/bin/python
	import sys
	import logging
	logging.basicConfig(stream=sys.stderr)
	sys.path.insert(0,"/var/www/FlaskApp/")

	from FlaskApp import app as application
	application.secret_key = 'Add your secret key''
  
```

To setup a virtual host file: `sudo nano /etc/apache2/sites-available/FlaskApp.conf`:
```
Virtual Host file
<VirtualHost *:80>
		ServerName 35.170.69.46
		ServerAdmin kirtivardhan.rai25995@gmail.com
		WSGIScriptAlias / /var/www/FlaskApp/flaskapp.wsgi
		<Directory /var/www/FlaskApp/FlaskApp/>
			Order allow,deny
			Allow from all
		</Directory>
		Alias /static /var/www/FlaskApp/FlaskApp/static
		<Directory /var/www/FlaskApp/FlaskApp/static/>
			Order allow,deny
			Allow from all
		</Directory>
		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log combined
	</VirtualHost>
  
  ```
  
### Install git, clone and setup your Item Catalog App project.
1. Install Git using `sudo apt-get install git`
2. Use `cd /var/www` to move to the /var/www directory 
3. Create the application directory `sudo mkdir FlaskApp`
4. Move inside this directory using `cd FlaskApp`
5. Clone the Catalog App to the virtual machine `git clone https://github.com/rambo255/item_catalog.git FlaskApp`
6. Move to the inner FlaskApp directory using `cd FlaskApp`
7. Rename `website.py` to `__init__.py` using `sudo mv website.py __init__.py`

### Flask
Do these commands:

```
sudo apt-get install python-pip python-flask python-sqlalchemy python-psycopg2
sudo pip install oauth2client requests httplib2
```

## Install PostGreSql
Install PostGreSql `sudo apt-get install postgresql`
To make sure remote are not allowed, please check the following configuration file: `sudo nano /etc/postgresql/9.5/main/pg_hba.conf`

### Create database
`sudo su - postgres`
Type in `psql` as postgres user

Do following the commands:
```
postgres=# CREATE USER catalog WITH PASSWORD 'catalog';
postgres=# ALTER USER catalog CREATEDB;
postgres=# CREATE DATABASE catalog WITH OWNER catalog;
```
Connect to the catalog database: `\c catalog`
```
catalog=# REVOKE ALL ON SCHEMA public FROM public;
catalog=# GRANT ALL ON SCHEMA public TO catalog;
```
Exit postgres and postgres user
```
postgres=# \q
postgres@PublicIP~$ exit
```

We need to update our database setup and application python files to illustrate the new engine connection
`engine = create_engine('postgresql://catalog:catalog@localhost/catalog')`

Run `sudo python database_setup.py`

## Oauth Client Login


* To get the Google+ authorization working:
    * Go to the project on the Developer Console: https://console.developers.google.com/project
    * Navigate to APIs & auth > Credentials > Edit Settings
    * add your host name and public IP-address to your Authorized JavaScript origins and your host name + oauth2callback to           Authorized redirect URIs, e.g. http://ec2-35-154-108-134.ap-south-1.compute.amazonaws.com/oauth2callback
    * Update data-client id in `templates/login.html`

Update the path in the __init__.py program. Update the client id and oauth_flow

```
CLIENT_ID = json.loads(
    open('/var/www/itemsCatalog/vagrant/catalog/client_secrets.json', 'r').read())['web']['client_id']

oauth_flow = flow_from_clientsecrets('/var/www/itemsCatalog/vagrant/catalog/client_secrets.json', scope='')
```
* restart apache
        * `sudo service apache2 restart`


## References:
https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
