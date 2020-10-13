# WP-Server-Installation
Following is a tutorial last updated on **October 10, 2020** to install a Latest configuration **WordPress** Server with latest of **Nginx, MariaDB, PHP-7.4, Redis & Let'sEncrypt SSL certbot** on **Ubuntu 20.04 LTS** virtual machine.

To try this Tutorial you can get a Virtual Server from **DigitalOcean**

[Get a $100 credit to try using our affiliate link](https://m.do.co/c/07aaa6a113b1) 

or create an Account on [DigitalOcean](https://digitalocean.com/)

Once you have your virtual machine provisioned, let's start
## 1. SSH onto the droplet

ssh root@[ip of the droplet]

## 2. Upgrade the new instance

Since most new instances are created from an image, the software in the image is likely behind so the first thing to do is to update it all. 

`$ apt-get update`

`$ apt-get upgrade (confirm if asked)`

## 3. Create swap

This step is important so that if any of our programs runs out of memory, it can use swap. Not creating it might lead to all kinds of problems with system instability. Although DigitalOcean tutorials recommends _not_ using swap on SSD, it is still useful as a protection. It should be always empty, so if it is not, strongly consider increasing the Droplet size.

```
$ fallocate -l 1G /swapfile
$ chmod 0600 /swapfile
$ mkswap /swapfile
$ swapon /swapfile
$ echo "/swapfile none swap sw 0 0" >> /etc/fstab
$ cat /proc/meminfo 
```

## 4. Install basic components the system will need

`$ apt-get install -y git curl apt-transport-https`

## 5. Install Nginx Server 
Nginx server is available in the Ubuntu repo but it is _not_ the latest version.😏

Currently latest Nginx version is 1.18.0 in stable.

To install latest version for Ubuntu 20.04; it is recommended to refer instructions from site - http://nginx.org/en/linux_packages.html#Ubuntu

Install the prerequisites:

`$ sudo apt install curl gnupg2 ca-certificates lsb-release`

To set up the apt repository for stable nginx packages, run the following command

`$ echo "deb http://nginx.org/packages/ubuntu `lsb_release -cs` nginx" \`
`	        | sudo tee /etc/apt/sources.list.d/nginx.list`

Next, import an official nginx signing key so apt could verify the packages authenticity:

`$ curl -fsSL https://nginx.org/keys/nginx_signing.key | sudo apt-key add -`

Verify that you now have the proper key:

`$ sudo apt-key fingerprint ABF5BD827BD9BF62`

The output should contain the full fingerprint 573B FD6B 3D8F BC64 1079 A6AB ABF5 BD82 7BD9 BF62 as follows:
```
pub   rsa2048 2011-08-19 [SC] [expires: 2024-06-14]
	          573B FD6B 3D8F BC64 1079  A6AB ABF5 BD82 7BD9 BF62
	    uid   [ unknown] nginx signing key <signing-key@nginx.com>
```
To install nginx, run the following commands:

`$ sudo apt update`
`$ sudo apt install nginx`

Check installed version 
	
`$ nginx -v `

Gives output
	
`nginx version: nginx/1.18.0`

Latest Stable Nginx is installed now 👍

## 6. Installing PHP 7.4 

Include Ondrej Repo for latest PHP 7.4 packages

Repo page is - https://launchpad.net/~ondrej/+archive/ubuntu/php

`$ sudo add-apt-repository ppa:ondrej/php`

`$ sudo apt update`

`$ sudo apt-get install php7.4-cli php7.4-curl php7.4-mysql php7.4-fpm php7.4-gd php7.4-xml php7.4-mbstring php7.4-zip php7.4-soap php7.4-dev `

## 7. Update Nginx to work with PHP-7.4

First, edit the default nginx configuration: 

Default nginx configuration is stored in **/etc/nginx/nginx.conf** file

I use nano over vim, not a nerd yet🤓 

`$ nano /etc/nginx/nginx.conf`

Here, change user to www-data; and worker_processes 1 to auto;

`$ nano /etc/nginx/conf.d/default.conf`

	Here, 
		change to server_name _;,
	then 
		in location / block, add index.php 
		so that the PHP files are read by Nginx. 

	Then 
		uncomment the second location ~ \.php$ block which include fastcgi_pass & fastcgi_param
		and change 

		fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;

	Also, in this block, change the root /usr/share/nginx/html.

	First check whether Nginx configuration is in order, then reload it:

	`$ sudo service nginx configtest`

	If the result is OK reload it, else rectify the error, often spelling mistakes

	`$ sudo service nginx reload`

	Lets configure the PHP7.4 FPM port from which it will connect with Nginx
	
	`$ cd /etc/php/7.4/fpm`
	
	`$ nano ./pool.d/www.conf`

	Around line 36, there should be a listen directive, which we need to replace with listen = [::]:9000.

	Now reload PHP-FPM: 

	`$ service php7.4-fpm reload`


	Visit the site root 
	`$ cd /usr/share/nginx/html`

	Create a phpinfo file there
	`$ nano info.php`

	In the file put phpinfo statement as

	<?php phpinfo(); ?>

	Ctrl + O to write the file & confirm

	Open your browser and type address as
	<your ip>/info.php

	the phpinfo(); should be revealed. Afterwards, remove it for safety reason.

	As PHPINFO is visible Nginx is talking to PHP7.4-FPM properly and now we can setup remaining parts i.e. REDIS, MYSQL, PMA & HTTPS

	In case of error do check the error log
	
	`$ cat /var/log/nginx/error.log`

	This log can help locate the errors easily.

## 8. Installing REDIS
	Redis is an in-memory key-value store known for its flexibility, performance, and wide language support. 
	We use REDIS to support Caching for WordPress

	Let's check if Ubuntu repo has latest version of Redis server
	
	`$ sudo apt-cache policy redis-server`

	Also visit redis.io to know latest available version. 
	
	Ubuntu 20.04 LTS cache has v5.0.7 while latest is v6.0.6, we cannot directly install latest redis, we need to find a repo for it
	Redis repo by Chris Lea has latest version (https://launchpad.net/~chris-lea/+archive/ubuntu/redis-server)


	`$ sudo add-apt-repository ppa:chris-lea/redis-server`
	`$ sudo apt update`
    `$ sudo apt install redis-server`

    This will install Redis server & dependencies.

    Let's check the version

    `$ redis-server -v`

    You will get the installed version number for redis-server

    Next, we have to configure few items

    sudo nano /etc/redis/redis.conf

    In the file, find the supervised directive. By Default it is set to no. 
    As we are running an operating system that uses the systemd init system, 
    we can change this to systemd

    supervised systemd

    Further scrolling down to the file get dir path

    it should be dir /var/lib/redis
    if not, comment older path & create the same.

    Also add
    maxmemory 256mb
	maxmemory-policy allkeys-lru

	Now lets restart the service

	`$ service redis-server restart`

	Now we will need PHP extension php-redis for using it in WordPress or any PHP application

	`$ sudo apt install php-redis`

	Lets enable it in case you have changed your PHP version in between

	`$ sudo phpenmod -v 7.4 -s ALL redis`

## 9. Install & setup MySQL

	We have options between MySQL official version (not recommended), Percona or MariaDB.
	I prefer MariaDB for it's truly open-source license & better performance.

	Ubuntu cache has slightly older release of 10.3.22 
	Instead let's follow offical way to get latest release [here](https://downloads.mariadb.org/mariadb/repositories/) 

	`$ sudo apt-get install software-properties-common`

	Let's fetch the key
	`$ sudo apt-key adv --fetch-keys 'https://mariadb.org/mariadb_release_signing_key.asc'`

	Add Repo - you can choose between available mirrors, this one is near for my DC
	`$ sudo add-apt-repository 'deb [arch=amd64,arm64,ppc64el] http://mirror2.hs-esslingen.de/mariadb/repo/10.5/ubuntu focal main'`

	Refresh the repos
	`$ sudo apt update`

	Install MariaDB
	`$ sudo apt install mariadb-server`

	As installation is done
	its time to secure the mysql-server

	`$ mysql_secure_installation`

	Create a new Root password for this
	It will ask for updating root password it is recommended to update it

	Give 'n' to unix_socket authentication
	Give 'y' to Change the root password
	Add New root password 

	Remove test database and access to it? [Y/n] y
	
	Reload privilege tables now? [Y/n] y
	
	All done!  If you've completed all of the above steps, your MariaDB
	installation should now be secure.


	Check service status 
	`$ sudo service mysql status`

	In case service is inactive start it as

	`$ sudo service mysql start`

## 10. Updating Nginx root to production folder
	I am planning to host atleast 3 websites here, so I am moving the nginx root to fairly standard directory /var/www/

	Current Nginx root /usr/share/nginx/html

	For Production purpose let's change it to /var/www/

	Create the directory 

	`$ mkdir /var/www`

	Let's update Nginx root in the default config (note - info.php won't be available now as root has been changed)

	`$ nano /etc/nginx/conf.d/default.conf`

	Comment previous root tags at two places & add new path as 
		root /var/www;

		1. At location / 
		2. At location ~ \.php$ 

	Configtest
	`$ sudo service nginx configtest`

	Reload to make new configs live

	`$ sudo service nginx reload`

	All should go well here, in case not, check for spelling mistakes or actual paths availability.

	Create a test.html file as follows

		<html>
			<head>
				<title>
					Testing new config
				</title>
			</head>
			<body>
				<h1>
					Test successful
				</h1>
			</body>
		</html>

	Open your browser & navigate to <your ip>/test.html

	It should give you test successful message.


## 11. Installing PhpMyAdmin for easier database management
	(REF - https://devanswers.co/installing-phpmyadmin-nginx-ubuntu-18-04/)

	It is fairly straightforward installation, but requires some additional configuration as we are using Nginx, which isn't natively supported in PhpMyAdmin standard configs.

	`$ sudo apt-get update`
	`$ sudo apt-get install phpmyadmin`

	When you are prompted to choose a web server like below, as there is no option for Nginx, press TAB and then ENTER to continue without selecting a web server.

	Next window will be for configuring phpmyadmin
	Select yes

	MySQL application password for phpmyadmin it is used internally by PMA to communication with MySQL - leave it blank and it will be generated automatically

	next it will ask for password for database administrative user - enter MySQL root password there.

	Now is the Nginx specific part

	In order for Nginx to serve the phpMyAdmin files correctly, we must create a symbolic link from the phpMyAdmin directory /usr/share/phpmyadmin to the Nginx document root directory, which we changed in last step.
	
	Lets create symlink for PhpMyAdmin

	`$ sudo ln -s /usr/share/phpmyadmin /var/www/phpmyadmin`
	`$ sudo service nginx reload`

	Now navigate to <your ip>/phpmyadmin 

	You should be able to see phpMyAdmin homepage
	Login with either root account or create a new account for PMA access

	to create new user

	Login to MySQL

	`$ sudo mysql -p -u root`

	Create new user
	CREATE USER 'pmauser'@'%' IDENTIFIED BY 'password_here';
	
	Lets make it superuser 
	GRANT ALL PRIVILEGES ON *.* TO 'pmauser'@'%';

	Reload the grant tables
	FLUSH PRIVILEGES;
	
	exit; 

	to exit from MySQL
	
	This should be good to access PMA over browser now

	Lets Secure PHPMyadmin by HTTPAuth

	We will need apache2-utils, which can generate .htpasswd file that works with both Nginx & Apache

	`$ sudo apt install apache2-utils`

	Once installed, we can generate the .htpasswd file.

	`$ sudo htpasswd -c /etc/nginx/.htpasswd securedusername`

	Replace securedusername with something sercured.
	It will ask you to enter new password
	Generate a password and keep it safe.
	
	Lets check if the command worked.

	`$ cat /etc/nginx/.htpasswd`
	You will get username & encrypted password

	Lets apply it for phpmyadmin

	Edit /etc/nginx/conf.d/default.conf

	Add a new block
		location /phpmyadmin {
		root /var/www;
		index index.php index.htm index.html;
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/.htpasswd;
		}

## 12. Let's map our first website here
	WordPress will be installed in /var/www/wordpress folder

	But will start with nginx block 
	Older nginx had sites-available and sites-enabled folders, but with new config its all within conf.d folder.
	as nginx.conf refers as follows
		 include /etc/nginx/conf.d/*.conf;
	we have two ways to approach this
	
	1. Create individual site conf file e.g. mysite.com.conf in /etc/nginx/conf.d/custom
	
	or
	2. Create sites-available & sites-enabled folders manually and importing those configus in /etc/nginx/nginx.conf as follows

	include /etc/nginx/sites-enabled/*.conf;

	Either way, we will need to create individual site conf file just the directory location varies. 

	I am using method 1 here.

	create a file mysite.com.conf in /etc/nginx/conf.d/ directory

	File contents as follows
		Server block code is in conf file block.txt file

## 13. Configuring UFW 
	Ubuntu FireWall UFW is available on server, but just inactive.

	Lets have OpenSSH whitelisted first

	`$ sudo ufw allow OpenSSH`

	After confirmation of Rules Updated let's enable it

	`$ sudo ufw enable`
	If will give you warning about disruption in existing SSH connection, but we have whitelisted that above. So press 'y'

	`$ sudo ufw status`

	Output will be in this line
	
	Status: active

	To                         Action      From
	--                         ------      ----
	OpenSSH                    ALLOW       Anywhere
	OpenSSH (v6)               ALLOW       Anywhere (v6)

	Now webserver need to be whitelisted.

	Standard command for it is
	`$ sudo ufw allow 'Nginx Full'`

	But sometimes Nginx profile is not available and you get an error.
	You can see which apps are available by running this command:

	`$ ufw app list`

	If you are wanting to accomplish this via application you will need to create the application ini file within /etc/ufw/applications.d

	Example:

	`$ nano /etc/ufw/applications.d/nginx.ini`

	Place this inside file

	[Nginx HTTP]
	title=Web Server 
	description=Enable NGINX HTTP traffic
	ports=80/tcp

	[Nginx HTTPS] \
	title=Web Server (HTTPS) \
	description=Enable NGINX HTTPS traffic
	ports=443/tcp

	[Nginx Full]
	title=Web Server (HTTP,HTTPS)
	description=Enable NGINX HTTP and HTTPS traffic
	ports=80,443/tcp

	Save the file & close editor
	
	Let's update the config

	`$ ufw app update nginx`

	Now app list will be updated. Check it
	
	`$ ufw app list`
	
	Output should show you Nginx options along with OpenSSH
	
	Available applications:
	  Nginx Full
	  Nginx HTTP
	  Nginx HTTPS
	  OpenSSH
	
	Individual config info is available as

	`$ ufw app info 'Nginx HTTP'`
	
	Profile: Nginx HTTP
	Title: Web Server
	Description: Enable NGINX HTTP traffic

	Port:
	  80/tcp
	
	As we are yet to setup HTTPS, start with HTTP

	`$ ufw allow 'Nginx HTTP'`
	Rule added
	Rule added (v6)

## 14. Install Certbot for Let'sEncrypt SSL
	
	`$ sudo apt install certbot python3-certbot-nginx`

	Before we run certbot firewall need to whitelist the traffic

	`$ sudo ufw allow 'Nginx Full'`
    `$ sudo ufw delete allow 'Nginx HTTP'`

    Confirm the status

    `$ sudo ufw status`
	
	Status: active

	To                         Action      From
	--                         ------      ----
	OpenSSH                    ALLOW       Anywhere
	Nginx Full                 ALLOW       Anywhere
	OpenSSH (v6)               ALLOW       Anywhere (v6)
	Nginx Full (v6)            ALLOW       Anywhere (v6)

	Obtaining an SSL Certificate

	Certbot has the Nginx plugin, which will take care of reconfiguring Nginx and reloading the config whenever necessary:

    `$ sudo certbot --nginx -d 1learn.in -d www.1learn.in`

    If this was first time running certbot on your server, it will prompt for email address and some more info choices.
    Pleaes make sure your domain points to the IP of current VM before you go for this.
    Once Certbot challenges are successful, it will as for HTTPS settings redirect options, 1. No Redirect or 2. Redirect all requests to HTTPS access.

    Select 2 and ENTER.

    Certbot will install the certificate & reconfigure Nginx to serve it directly.

    But these certificates are valid for 90 days only, there we require Auto-Renewal.
    Certbot nginx plugin has autorenewal configured, just we need to verify that it is added in the cron

    Visit cron directory

    `$ cd /etc/cron.d`

    There you will find a file certbot, if interested, open it & check the config.

    $nano certbot

    SHELL=/bin/sh
	PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

	0 */12 * * * root test -x /usr/bin/certbot -a \! -d /run/systemd/system && perl>

	This makes sure that cron will check for any certificate which will be due for renewal and process the renewal.

	Also you can check by dry-run

	`$ sudo certbot renew --dry-run`
	If it gives successful output, your configuration is properly placed.

15. Installing WP-CLI
