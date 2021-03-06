---
layout: post
title:  "Instalación de Nagios en Ubuntu 14.04 LTS"
date:   2015-06-17 21:26:44
categories: nagios
---

# Requisitos previos
Para poder seguir este tutorial, hay que tener unos requisitos básicos. 
    - Tener acceso a privilegios de superusuario ya sea con sudo o con root. 
    - En nuestro caso hemos usado 1 core, 1GB RAM y 20GB HDD
    - Tener instalado LAMP y SSH server

# Crear el usuario Nagios y su grupo

{% highlight bash %}
sudo useradd nagios
sudo groupadd nagcmd
sudo usermod -a -G nagcmd nagios
{% endhighlight %}

# Instalar Dependencias de la instalación

Debido a que estamos instalando Nagios Core desde el codigo fuente, necesitaremos instalar unas cuantas librerias de desarrollador para completar la instalación, además necesitaremos instalar ``apache2-utils``, con la que se instalará la interfaz web de Nagios.
Pero primero, hagamos una actualización de la lista de paquetes:

{% highlight bash %}
sudo apt-get update
{% endhighlight %}

Y ahora instalamos los paquetes requeridos:

{% highlight bash %}
sudo apt-get install build-essential sendemail libssl-dev apache2-utils xinetd libgd2-xpm-dev unzip snmp
{% endhighlight %}

Ya podemos instalar Nagios.

# Instalar Nagios Core

{% highlight bash %}
cd ~
wget http://prdownloads.sourceforge.net/sourceforge/nagios/nagios-4.1.1.tar.gz
{% endhighlight %}


Estraemos los archivos de nagios:

{% highlight bash %}
tar xvf nagios-*.tar.gz
{% endhighlight %}

Y cambiamos al directorio que acabamos de extraer

{% highlight bash %}
cd nagios-*
{% endhighlight %}


Antes de instalar Nagios, debemos de configurarlo. Si quieres configurarlo para usar postfix,sendmail o otro (puedes descargarlo con apt-get o aptitude) añade 
``-–with-mail=/usr/sbin/sendmail`` al siguiente comando:

{% highlight bash %}
./configure --with-nagios-group=nagios --with-command-group=nagcmd --with-mail=/usr/sbin/sendmail
{% endhighlight %}
Nota: Nosotros configuramos los correos con sendemail en vez de sendmail ya que es mucho más liviano y consume menos recursos. Pero en la configuración ponemos sendmail para que genere el comando de correo que posteriormente modificaremos

Ahora compilamos Nagios con el comando:

{% highlight bash %}
sudo make all
sudo make install
sudo make install-init
sudo make install-config
sudo make install-commandmode
sudo /usr/bin/install -c -m 644 sample-config/httpd.conf /etc/apache2/sites-available/nagios.conf
{% endhighlight %}

Para poder ejecutar comandos externos por la interfaz web de NAgios, debemos añadir al usuario de Apache ``www-data`` al grupo ``nagcmd`` para que tenga los permisos suficientes:

{% highlight bash %}
sudo usermod -G nagcmd www-data
{% endhighlight %}

#Crear el usuario para loguear en nagios y asignarle contraseña
{% highlight bash %}
sudo htpasswd -cm /usr/local/nagios/etc/htpasswd.users nagiosadmin
New Password: *********
Re-type new password: *********
{% endhighlight %}

# Congigurando Apache

Activamos los modulos de apache rewrite y cgi:

{% highlight bash %}
sudo a2enmod rewrite
sudo a2enmod cgi
{% endhighlight %}

Usaremos htpasswd para crear un administrador de usuario, llamado "nagiosadmin", que tendrá el acceso a la interface web de Nagios:

{% highlight bash %}
sudo htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
{% endhighlight %}

Escribe la password. Recuerdala por que la necesitaras para poder acceder a Nagios desde la web.
Ahora crearemos un enlace simbolico de ``nagios.conf`` a el directorio ``sites-enabled``:

{% highlight bash %}
sudo a2ensite nagios.conf
{% endhighlight %}

Ahora nagios está listo para arrancar. Arranquemoslo y también Apache (para poder verlo):

{% highlight bash %}
sudo service nagios restart
sudo service apache2 restart
{% endhighlight %}

# Accediendo a la interfaz web de Nagios
Abre el navegador y abre tu servidor nagios (sustituye la ip o el nombre del equipo por el tuyo)

{% highlight bash %}
http://nagios_server_ip/nagios
{% endhighlight %}

Debido a que se ha configurado apache para usar htpasswd, debes introducir el usuario y contraseña que hemos creado antes. Recuerda que usamos "nagiosadmin" como usuario

<img src="https://jedelwey.github.io/images/autenticacion.PNG"/>


Después de autenticarte, puedes ver la página por defecto de Nagios. Ahora haga click en Hosts en la parte izquierda de la web para ver los equipos que estás monitorizando.

<img src="https://jedelwey.github.io/images/nagios-core.PNG"/>

# Nagios en el arranque del sistema
Para arrancar Nagios al arrancar el servidor, crearemos un enlace simbolico:

{% highlight bash %}
sudo ln -s /etc/init.d/nagios /etc/rcS.d/S99nagios
{% endhighlight %}

Como puedes ver funciona perfectamente.

# Instalar NRPE
Descargamos NRPE y lo descomprimimos
{% highlight bash %}
wget http://sourceforge.net/projects/nagios/files/nrpe-2.x/nrpe-2.15/nrpe-2.15.tar.gz
tar xzf nrpe-*.tar.gz
cd nrpe-*/
{% endhighlight %}

Compilamos el NRPE y lo instalamos
{% highlight bash %}
./configure --with-ssl=/usr/bin/openssl --with-ssl-lib=/usr/lib/x86_64-linux-gnu 
sudo make all 
sudo make install-plugin 
sudo make install-daemon 
sudo make install-daemon-config
{% endhighlight %}

Instalamos el demonio NRPE como un servicio bao xinetd
{% highlight bash %}
make install-xinetd
{% endhighlight %}

Editamos /etc/xinetd.d/nrpe y añadimos la IP del servidor donde esta instalado el Nagios:

{% highlight bash %}
only_from = 127.0.0.1,Servidor_Nagios_IP
{% endhighlight %}
Añadimos la siguiente entrada para el NRPE en el /etc/services:
{% highlight bash %}
nrpe            5666/tcp                        # NRPE
{% endhighlight %}
Reiniciamos el xinetd y la red:
{% highlight bash %}
sudo service xinetd restart # service networking restart
{% endhighlight %}
Comprobamos que este funcionando correctamente:

{% highlight bash %}
netstat -at | grep nrpe tcp 0 0 *:nrpe *:* LISTEN
{% endhighlight %}

Comprobamos que funcione correctamente el NRPE:

{% highlight bash %}
sudo /usr/local/nagios/libexec/check_nrpe -H localhost
{% endhighlight %}
Que nos debería mostar: 
NRPE v2.15 o la versión que tengamos instalado
# Instalar Nagios Plugins

{% highlight bash %}
cd ~
wget http://nagios-plugins.org/download/nagios-plugins-2.1.1.tar.gz
{% endhighlight %}

Estraemos los archivos de Nagios Plugins con el comando:

{% highlight bash %}
tar xvf nagios-plugins-*.tar.gz
{% endhighlight %}

Nos cambiamos a la carpeta de Nagios plugins:

{% highlight bash %}
cd nagios-plugins-*
{% endhighlight %}

Como con Nagios Core, antes debemos configurarlo. Para ello escribimos: 

{% highlight bash %}
./configure --with-nagios-user=nagios --with-nagios-group=nagios --with-openssl
{% endhighlight %}

Ahora compilamos con el comando:

{% highlight bash %}
sudo make
{% endhighlight %}

Y lo instalamos con este otro:

{% highlight bash %}
sudo make install
{% endhighlight %}

Ahora reiniciamos apache para que hagan cambio los efectos:

{% highlight bash %}
sudo service nagios stop
sudo service nagios start
sudo service apache2 stop
sudo service apache2 start
{% endhighlight %}

Nagios está ahora funcionando, vamos a entrar y iniciar sesión en él.
