---
layout: post
title:  "Instalación de Nagios en Ubuntu 14.04 LTS"
date:   2015-06-17 21:26:44
categories: nagios
image: ../images/nagios-core.PNG
---

# Requisitos previos
Para poder seguir este tutorial, hay que tener unos requisitos básicos. 
    - Tener acceso a privilegios de superusuario ya sea con sudo o con root. 
    - Tener instalado LAMP (Linux, Apache, MySQL, PHP)
    - Tener al menos 2 GB de memoria Swap

# Crear el usuario Nagios y su grupo
{% highlight bash %}
sudo useradd nagios
sudo groupadd nagcmd
sudo usermod -a -G nagcmd nagios
{% endhighlight %}
# Install Build Dependencies

Because we are building Nagios Core from source, we must install a few development libraries that will allow us to complete the build. While we're at it, we will also install apache2-utils, which will be used to set up the Nagios web interface.

First, update your apt-get package lists:
{% highlight bash %}
sudo apt-get update
{% endhighlight %}
Then install the required packages:
{% highlight bash %}
sudo apt-get install build-essential libgd2-xpm-dev openssl libssl-dev xinetd apache2-utils 
{% endhighlight %}
Let's install Nagios now.

# Install Nagios Core

Download the source code for the latest stable release of Nagios Core. Go to the [Nagios-core][Nagios downloads page], and click the Skip to download link below the form. Copy the link address for the latest stable release so you can download it to your Nagios server.

At the time of this writing, the latest stable release is Nagios 4.0.8. Download it to your home directory with curl:
{% highlight bash %}
cd ~
curl -L -O http://prdownloads.sourceforge.net/sourceforge/nagios/nagios-4.0.8.tar.gz
{% endhighlight %}
Extract the Nagios archive with this command:
{% highlight bash %}
tar xvf nagios-*.tar.gz
{% endhighlight %}
Then change to the extracted directory:
{% highlight bash %}
cd nagios-*
{% endhighlight %}
Before building Nagios, we must configure it. If you want to configure it to use postfix (which you can install with apt-get), add -–with-mail=/usr/sbin/sendmail to the following command:
{% highlight bash %}
./configure --with-nagios-group=nagios --with-command-group=nagcmd 
{% endhighlight %}
Now compile Nagios with this command:
{% highlight bash %}
make all
{% endhighlight %}
Now we can run these make commands to install Nagios, init scripts, and sample configuration files:
{% highlight bash %}
sudo make install
sudo make install-commandmode
sudo make install-init
sudo make install-config
sudo /usr/bin/install -c -m 644 sample-config/httpd.conf /etc/apache2/sites-available/nagios.conf
{% endhighlight %}
In order to issue external commands via the web interface to Nagios, we must add the web server user, www-data, to the nagcmd group:
{% highlight bash %}
sudo usermod -G nagcmd www-data
{% endhighlight %}
# Install Nagios Plugins

Find the latest release of Nagios Plugins here: [Nagios Plugins Download][Nagios-plugins]. Copy the link address for the latest version, and copy the link address so you can download it to your Nagios server.

At the time of this writing, the latest version is Nagios Plugins 2.0.3. Download it to your home directory with curl:
{% highlight bash %}
cd ~
curl -L -O http://nagios-plugins.org/download/nagios-plugins-2.0.3.tar.gz
{% endhighlight %}
Extract Nagios Plugins archive with this command:
{% highlight bash %}
tar xvf nagios-plugins-*.tar.gz
{% endhighlight %}
Then change to the extracted directory:
{% highlight bash %}
cd nagios-plugins-*
{% endhighlight %}
Before building Nagios Plugins, we must configure it. Use this command:
{% highlight bash %}
./configure --with-nagios-user=nagios --with-nagios-group=nagios --with-openssl
{% endhighlight %}
Now compile Nagios Plugins with this command:
{% highlight bash %}
make
{% endhighlight %}
Then install it with this command:
{% highlight bash %}
sudo make install
{% endhighlight %}
# Install NRPE

Find the source code for the latest stable release of NRPE at the [NRPE downloads page][NRPE]. Download the latest version to your Nagios server.

At the time of this writing, the latest release is 2.15. Download it to your home directory with curl:
{% highlight bash %}
cd ~
curl -L -O http://downloads.sourceforge.net/project/nagios/nrpe-2.x/nrpe-2.15/nrpe-2.15.tar.gz
{% endhighlight %}
Extract the NRPE archive with this command:
{% highlight bash %}
tar xvf nrpe-*.tar.gz
{% endhighlight %}
Then change to the extracted directory:
{% highlight bash %}
cd nrpe-*
{% endhighlight %}
Configure NRPE with these commands:
{% highlight bash %}
./configure --enable-command-args --with-nagios-user=nagios --with-nagios-group=nagios --with-ssl=/usr/bin/openssl --with-ssl-lib=/usr/lib/x86_64-linux-gnu
{% endhighlight %}
Now build and install NRPE and its xinetd startup script with these commands:
{% highlight bash %}
make all
sudo make install
sudo aptitude install xinetd
sudo make install-xinetd
sudo make install-daemon-config
{% endhighlight %}
Open the xinetd startup script in an editor:
{% highlight bash %}
sudo nano /etc/xinetd.d/nrpe
{% endhighlight %}
Modify the only_from line by adding the private IP address of the your Nagios server to the end (substitute in the actual IP address of your server):
{% highlight bash %}
only_from = 127.0.0.1 192.168.1.200
{% endhighlight %}
Save and exit. Only the Nagios server will be allowed to communicate with NRPE.

Restart the xinetd service to start NRPE:
{% highlight bash %}
sudo service xinetd restart
{% endhighlight %}
Now that Nagios 4 is installed, we need to configure it.

# Configure Nagios
Now let's perform the initial Nagios configuration. You only need to perform this section once, on your Nagios server.

# Organize Nagios Configuration

Open the main Nagios configuration file in your favorite text editor. We'll use nano to edit the file:
{% highlight bash %}
sudo nano /usr/local/nagios/etc/nagios.cfg
{% endhighlight %}
Now find an uncomment this line by deleting the #:

#cfg_dir=/usr/local/nagios/etc/servers
Save and exit.

Now create the directory that will store the configuration file for each server that you will monitor:
{% highlight bash %}
sudo mkdir /usr/local/nagios/etc/servers
{% endhighlight %}
# Configure Nagios Contacts

Open the Nagios contacts configuration in your favorite text editor. We'll use nano to edit the file:
{% highlight bash %}
sudo nano /usr/local/nagios/etc/objects/contacts.cfg
{% endhighlight %}
Find the email directive, and replace its value (the highlighted part) with your own email address:
{% highlight bash %}
email                           nagios@finode.com        ; <<***** CHANGE THIS TO YOUR EMAIL ADDRESS ******
{% endhighlight %}
Save and exit.

#Configure check_nrpe Command

Let's add a new command to our Nagios configuration:
{% highlight bash %}
sudo nano /usr/local/nagios/etc/objects/commands.cfg
{% endhighlight %}
Add the following to the end of the file:
{% highlight bash %}
define command{
        command_name check_nrpe
        command_line $USER1$/check_nrpe -H $HOSTADDRESS$ -c $ARG1$
}
{% endhighlight %}
Save and exit. This allows you to use the check_nrpe command in your Nagios service definitions.

#Configure Apache

Enable the Apache rewrite and cgi modules:
{% highlight bash %}
sudo a2enmod rewrite
sudo a2enmod cgi
{% endhighlight %}
Use htpasswd to create an admin user, called "nagiosadmin", that can access the Nagios web interface:
{% highlight bash %}
sudo htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
{% endhighlight %}
Enter a password at the prompt. Remember this password, as you will need it to access the Nagios web interface.

Now create a symbolic link of nagios.conf to the sites-enabled directory:
{% highlight bash %}
sudo ln -s /etc/apache2/sites-available/nagios.conf /etc/apache2/sites-enabled/
{% endhighlight %}
Nagios is ready to be started. Let's do that, and restart Apache:
{% highlight bash %}
sudo service nagios start
sudo service apache2 restart
{% endhighlight %}
To enable Nagios to start on server boot, run this command:
{% highlight bash %}
sudo ln -s /etc/init.d/nagios /etc/rcS.d/S99nagios
{% endhighlight %}
# Optional: Restrict Access by IP Address
If you want to restrict the IP addresses that can access the Nagios web interface, you will want to edit the Apache configuration file:
{% highlight bash %}
sudo nano /etc/apache2/sites-available/nagios.conf
{% endhighlight %}
Find and comment the following two lines by adding # symbols in front of them:
{% highlight bash %}
Order allow,deny
Allow from all
{% endhighlight %}
Then uncomment the following lines, by deleting the # symbols, and add the IP addresses or ranges (space delimited) that you want to allow to in the Allow from line:
{% highlight bash %}
#  Order deny,allow
#  Deny from all
#  Allow from 127.0.0.1
{% endhighlight %}
As these lines will appear twice in the configuration file, so you will need to perform these steps once more.

Save and exit.

Now restart Apache to put the change into effect:
{% highlight bash %}
sudo service nagios restart
sudo service apache2 restart
{% endhighlight %}
Nagios is now running, so let's try and log in.

# Accessing the Nagios Web Interface
Open your favorite web browser, and go to your Nagios server (substitute the IP address or hostname for the highlighted part):
{% highlight bash %}
http://nagios_server_ip/nagios
{% endhighlight %}
Because we configured Apache to use htpasswd, you must enter the login credentials that you created earlier. We used "nagiosadmin" as the username:

<img src="../images/autenticacion.PNG"/>
After authenticating, you will be see the default Nagios home page. Click on the Hosts link, in the left navigation bar, to see which hosts Nagios is monitoring:

<img src="../images/nagios-core.PNG"/>
As you can see, Nagios is monitoring only "localhost", or itself.

Let's monitor another host with Nagios!

[NRPE]:             http://sourceforge.net/projects/nagios/files/nrpe-2.x/
[Nagios-plugins]:   http://nagios-plugins.org/download/?C=M;O=D
[Nagios-core]:      https://www.nagios.org/download/core/