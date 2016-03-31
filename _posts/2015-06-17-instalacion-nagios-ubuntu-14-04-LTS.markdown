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
sudo apt-get install build-essential sendemail libssl-dev apache2-utils xinetd libgd2-xpm-dev unzip
{% endhighlight %}

Ya podemos instalar Nagios.

# Instalar Nagios Core
Descargaremos el codigo fuente de la última versión estable de Nagios Core. Para ello vamos a [Nagios-core][página web de Nagios], y clickeamos en descargar. Copia la dirección del enlace para poder descargarla desde tu servidor Nagios.

En nuestro caso, la última estable es la 4.1.1. Nos la descargaremos en nuestro directorio con curl o wget:

{% highlight bash %}
cd ~
curl -L -O http://prdownloads.sourceforge.net/sourceforge/nagios/nagios-4.1.1.tar.gz
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

In order to issue external commands via the web interface to Nagios, we must add the web server user, www-data, to the nagcmd group:
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

# Accediendo a la interfaz web de Nagios
Abre el navegador y abre tu servidor nagios (sustituye la ip o el nombre del equipo por el tuyo)

{% highlight bash %}
http://nagios_server_ip/nagios
{% endhighlight %}

Debido a que se ha configurado apache para usar htpasswd, debes introducir el usuario y contraseña que hemos creado antes. Recuerda que usamos "nagiosadmin" como usuario

<img src="https://jedelwey.github.io/images/autenticacion.PNG"/>


Después de autenticarte, puedes ver la página por defecto de Nagios. Ahora haga click en Hosts en la parte izquierda de la web para ver los equipos que estás monitorizando.

<img src="https://jedelwey.github.io/images/nagios-core.PNG"/>


# Configurando Nagios
Ahora vamos a crear la configuración inicial de Nagios. 

# Organizar la configuración de Nagios

Abrimos el fichero principal de configuración de nagios con nuestro editor:

{% highlight bash %}
sudo nano /usr/local/nagios/etc/nagios.cfg
{% endhighlight %}

Ahora descomentamos esta linea borrando el #:

{% highlight bash %}
#cfg_dir=/usr/local/nagios/etc/servers
{% endhighlight %}

Guardamos y salimos.

Ahora crearemos el directorio en el que se guardaran los ficheros de cada servidor que queramos monitorizar:

{% highlight bash %}
sudo mkdir /usr/local/nagios/etc/servers
{% endhighlight %}

# Configure Nagios Contacts

Abrimos el fichero de configuración de Nagios:

{% highlight bash %}
sudo nano /usr/local/nagios/etc/objects/contacts.cfg
{% endhighlight %}

Encontramos la directiva de email y la reemplazamos por nuestar dirección de email:

{% highlight bash %}
email                           nagios@finode.com        ; <<***** CHANGE THIS TO YOUR EMAIL ADDRESS ******
{% endhighlight %}

Guardamos y salimos.

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
sudo service nagios stop
sudo service apache2 start
{% endhighlight %}

Para arrancar Nagios al arrancar el servidor, crearemos un enlace simbolico:

{% highlight bash %}
sudo ln -s /etc/init.d/nagios /etc/rcS.d/S99nagios
{% endhighlight %}

Como puedes ver funciona perfectamente.

# Instalar NRPE
Instalamos por la linea de consola

{% highlight bash %}
sudo aptitude install nagios-nrpe-server nagios-nrpe-plugin
{% endhighlight %}

# Configurando el commando check_nrpe

Añadiremos un nuevo comando a nuestra configuración de nagios, para ello editamos el fichero:

{% highlight bash %}
sudo nano /usr/local/nagios/etc/objects/commands.cfg
{% endhighlight %}

Y añadimos la siguientes lineas al fichero:

{% highlight bash %}
define command{
        command_name check_nrpe
        command_line $USER1$/check_nrpe -H $HOSTADDRESS$ -c $ARG1$
}
{% endhighlight %}

Guardamos y cerrramos. Esto te permite usar check_nrpe en tus definiciones de servicios de Nagios.

# Instalar Nagios Plugins

Encuentra la última versión estable de Nagios Plugins aquí: [Descargar Nagios Plugins][Nagios-plugins]. Copia la dirección para poder descargarla desde el servidor nagios.
En nuestro caso, la última versión estable es Nagios Plugins 2.0.3. La descargamos en nuestro directorio home con curl:

{% highlight bash %}
cd ~
curl -L -O http://nagios-plugins.org/download/nagios-plugins-2.0.3.tar.gz
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
make
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

Nagios está ahora funcionando, vmaos a entrar y iniciar sesión en él.



[Nagios-plugins]:   http://nagios-plugins.org/download/?C=M;O=D
[Nagios-core]:      https://www.nagios.org/download/core/
