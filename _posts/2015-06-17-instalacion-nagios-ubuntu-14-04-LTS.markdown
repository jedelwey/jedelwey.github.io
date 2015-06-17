---
layout: post
title:  "Instalación de Nagios en Ubuntu 14.04 LTS"
date:   2015-06-17 21:26:44
categories: nagios
---

# Requisitos previos
Para poder seguir este tutorial, hay que tener unos requisitos básicos. 
    - Tener acceso a privilegios de superusuario ya sea con sudo o con root. 
    - Tener instalado LAMP (Linux, Apache, MySQL, PHP)
    - Tener al menos 2 GB de memoria Swap

# Crear el usuario Nagios y su grupo
sudo useradd nagios
sudo groupadd nagcmd
sudo usermod -a -G nagcmd nagios

# Install Build Dependencies

Because we are building Nagios Core from source, we must install a few development libraries that will allow us to complete the build. While we're at it, we will also install apache2-utils, which will be used to set up the Nagios web interface.

First, update your apt-get package lists:

sudo apt-get update
Then install the required packages:

sudo apt-get install build-essential libgd2-xpm-dev openssl libssl-dev xinetd apache2-utils 
Let's install Nagios now.

# Install Nagios Core

Download the source code for the latest stable release of Nagios Core. Go to the Nagios downloads page, and click the Skip to download link below the form. Copy the link address for the latest stable release so you can download it to your Nagios server.

At the time of this writing, the latest stable release is Nagios 4.0.8. Download it to your home directory with curl:

cd ~
curl -L -O http://prdownloads.sourceforge.net/sourceforge/nagios/nagios-4.0.8.tar.gz
Extract the Nagios archive with this command:

tar xvf nagios-*.tar.gz
Then change to the extracted directory:

cd nagios-*
Before building Nagios, we must configure it. If you want to configure it to use postfix (which you can install with apt-get), add -–with-mail=/usr/sbin/sendmail to the following command:

./configure --with-nagios-group=nagios --with-command-group=nagcmd 
Now compile Nagios with this command:

make all
Now we can run these make commands to install Nagios, init scripts, and sample configuration files:

sudo make install
sudo make install-commandmode
sudo make install-init
sudo make install-config
sudo /usr/bin/install -c -m 644 sample-config/httpd.conf /etc/apache2/sites-available/nagios.conf
In order to issue external commands via the web interface to Nagios, we must add the web server user, www-data, to the nagcmd group:

sudo usermod -G nagcmd www-data

# Install Nagios Plugins

Find the latest release of Nagios Plugins here: Nagios Plugins Download. Copy the link address for the latest version, and copy the link address so you can download it to your Nagios server.

At the time of this writing, the latest version is Nagios Plugins 2.0.3. Download it to your home directory with curl:

cd ~
curl -L -O http://nagios-plugins.org/download/nagios-plugins-2.0.3.tar.gz
Extract Nagios Plugins archive with this command:

tar xvf nagios-plugins-*.tar.gz
Then change to the extracted directory:

cd nagios-plugins-*
Before building Nagios Plugins, we must configure it. Use this command:

./configure --with-nagios-user=nagios --with-nagios-group=nagios --with-openssl
Now compile Nagios Plugins with this command:

make
Then install it with this command:

sudo make install

# Install NRPE

Find the source code for the latest stable release of NRPE at the NRPE downloads page. Download the latest version to your Nagios server.

At the time of this writing, the latest release is 2.15. Download it to your home directory with curl:

cd ~
curl -L -O http://downloads.sourceforge.net/project/nagios/nrpe-2.x/nrpe-2.15/nrpe-2.15.tar.gz
Extract the NRPE archive with this command:

tar xvf nrpe-*.tar.gz
Then change to the extracted directory:

cd nrpe-*
Configure NRPE with these commands:

./configure --enable-command-args --with-nagios-user=nagios --with-nagios-group=nagios --with-ssl=/usr/bin/openssl --with-ssl-lib=/usr/lib/x86_64-linux-gnu
Now build and install NRPE and its xinetd startup script with these commands:

make all
sudo make install
sudo aptitude install xinetd
sudo make install-xinetd
sudo make install-daemon-config

sudo nano /etc/xinetd.d/nrpe

Modify the only_from line by adding the private IP address of the your Nagios server to the end (substitute in the actual IP address of your server):

only_from = 127.0.0.1 10.132.224.168
Save and exit. Only the Nagios server will be allowed to communicate with NRPE.

Restart the xinetd service to start NRPE:

sudo service xinetd restart
Now that Nagios 4 is installed, we need to configure it.

# Configure Nagios
Now let's perform the initial Nagios configuration. You only need to perform this section once, on your Nagios server.

# Organize Nagios Configuration

Open the main Nagios configuration file in your favorite text editor. We'll use nano to edit the file:

sudo nano /usr/local/nagios/etc/nagios.cfg
Now find an uncomment this line by deleting the #:

#cfg_dir=/usr/local/nagios/etc/servers
Save and exit.

Now create the directory that will store the configuration file for each server that you will monitor:

sudo mkdir /usr/local/nagios/etc/servers

# Configure Nagios Contacts

Open the Nagios contacts configuration in your favorite text editor. We'll use nano to edit the file:

sudo nano /usr/local/nagios/etc/objects/contacts.cfg
Find the email directive, and replace its value (the highlighted part) with your own email address:

email                           nagios@finode.com        ; <<***** CHANGE THIS TO YOUR EMAIL ADDRESS ******
Save and exit.

#Configure check_nrpe Command

Let's add a new command to our Nagios configuration:

sudo nano /usr/local/nagios/etc/objects/commands.cfg
Add the following to the end of the file:

define command{
        command_name check_nrpe
        command_line $USER1$/check_nrpe -H $HOSTADDRESS$ -c $ARG1$
}
Save and exit. This allows you to use the check_nrpe command in your Nagios service definitions.

#Configure Apache

Enable the Apache rewrite and cgi modules:

sudo a2enmod rewrite
sudo a2enmod cgi
Use htpasswd to create an admin user, called "nagiosadmin", that can access the Nagios web interface:

sudo htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
Enter a password at the prompt. Remember this password, as you will need it to access the Nagios web interface.

Now create a symbolic link of nagios.conf to the sites-enabled directory:

sudo ln -s /etc/apache2/sites-available/nagios.conf /etc/apache2/sites-enabled/
Nagios is ready to be started. Let's do that, and restart Apache:

sudo service nagios start
sudo service apache2 restart
To enable Nagios to start on server boot, run this command:

sudo ln -s /etc/init.d/nagios /etc/rcS.d/S99nagios

# Optional: Restrict Access by IP Address
If you want to restrict the IP addresses that can access the Nagios web interface, you will want to edit the Apache configuration file:

sudo nano /etc/apache2/sites-available/nagios.conf
Find and comment the following two lines by adding # symbols in front of them:

Order allow,deny
Allow from all
Then uncomment the following lines, by deleting the # symbols, and add the IP addresses or ranges (space delimited) that you want to allow to in the Allow from line:

#  Order deny,allow
#  Deny from all
#  Allow from 127.0.0.1
As these lines will appear twice in the configuration file, so you will need to perform these steps once more.

Save and exit.

Now restart Apache to put the change into effect:

sudo service nagios restart
sudo service apache2 restart
Nagios is now running, so let's try and log in.

# Accessing the Nagios Web Interface
Open your favorite web browser, and go to your Nagios server (substitute the IP address or hostname for the highlighted part):

http://nagios_server_public_ip/nagios
Because we configured Apache to use htpasswd, you must enter the login credentials that you created earlier. We used "nagiosadmin" as the username:

htaccess Authentication Prompt

After authenticating, you will be see the default Nagios home page. Click on the Hosts link, in the left navigation bar, to see which hosts Nagios is monitoring:

Nagios Hosts Page

As you can see, Nagios is monitoring only "localhost", or itself.

Let's monitor another host with Nagios!








Check out the [Jekyll docs][jekyll] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll’s dedicated Help repository][jekyll-help].

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
