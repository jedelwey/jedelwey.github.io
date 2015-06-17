---
layout: post
title:  "Monitorizar un Host o Cliente Linux con NRPE"
date:   2015-06-18 00:26:00
categories: nagios
---

# Monitorizar un Host o cliente Linux con NRPE

En este post, expplicaré como añadir un host o cliente Linux a nuestro nagios para ser monitorizado. Solo tienes que repetir este post para cada equipo que se quiera monitorizar. 
Empezaremos por actualizar la lsita de paquetes:

{% highlight bash %}
sudo apt-get update
{% endhighlight %}

Ahora instalamos Nagios Plugins y NRPE:

{% highlight bash %}
sudo apt-get install nagios-plugins nagios-nrpe-server
{% endhighlight %}

Después, vamos a modificar el fichero de configuración de NRPE:

{% highlight bash %}
sudo nano /etc/nagios/nrpe.cfg
{% endhighlight %}

Encontramos la directiva ``allowed_host`` y añadimos la IP privada de nuestro Nagios, con una coma por delante para separar la otra dirección IP.

{% highlight bash %}
allowed_hosts=127.0.0.1,10.132.224.168
{% endhighlight %}

Guardamos y cerramos. Esta configuración NRPE para aceptar peticiones NRPE desde tu servidor Nagios, por esa dirección IP.
Reiniciamos NRPE para que los cambios hagan efecto:

{% highlight bash %}
sudo service nagios-nrpe-server restart
{% endhighlight %}

Una vez termines de instalar y configurar NRPE en el host o cliente que quieres monitorizar, debes añadir el host o cliente a tu servidor Nagios en los archivos de configuración para que se puedan comenzar a monitorizar.

Pero en el cliente esto es todo, ya hemos acabado.

# Añadir un Host o cliente a nuestra configuración de Nagios

En el servidor nagios crearemos un fichero de configuración por cada host o equipo que queramos monitorizar en ``/usr/local/nagios/etc/servidores`` ``/usr/local/nagios/etc/clientes`` ``/usr/local/nagios/etc/switches`` ``/usr/local/nagios/etc/routers`` ``/usr/local/nagios/etc/firewall``. Recuerda reemplazar ``tuequipo`` por el nombre del host o equipo que quieras monitorizar.

{% highlight bash %}
sudo nano /usr/local/nagios/etc/servers/yourhost.cfg
{% endhighlight %}

En el siguiente host o equipo remplazamos el campo ``host_name`` por el nombre de tu equipo o host ("web-1", "fw-1", etc), en el alias ponemos una descripción del equipo que nos permita identificarla y en la dirección ponemos la IP del equipo o host:

{% highlight bash %}
define host {
        use                             linux-server
        host_name                       tuequipo
        alias                           My first Apache server
        address                         10.132.234.52
        max_check_attempts              5
        check_period                    24x7
        notification_interval           30
        notification_period             24x7
}
{% endhighlight %}

Con la configuración de arriba, Nagios, solo podrá saber si el equipo está encendido o apagado. Si esto es suficiente para ti, guarda, cierra y reincia nagios. Si quieres monitorizar más servicios sigamos añadiendo.

Añade uno de estos servicios si quieres monitorizarlos. Como puedes comprobar el valor de ``check_command`` determina que es lo que se va a monitorizar, incluyendo los umbrales del servicio. Aquí tienes algunos ejemplos de configuración:

Ping:

{% highlight bash %}
define service {
        use                             generic-service
        host_name                       tuequipo
        service_description             PING
        check_command                   check_ping!100.0,20%!500.0,60%
}
{% endhighlight %}

SSH (notifications_enabled puesto a 0 desactiva las notificaciones para un servicio):

{% highlight bash %}
define service {
        use                             generic-service
        host_name                       yourhost
        service_description             SSH
        check_command                   check_ssh
        notifications_enabled           0
}
{% endhighlight %}

Si no estás seguro de lo que quiere monitorizar, use los servicios genericos, simplemente toma los valores de una plantilla ``generic-service`` que se crea por defecto.
Ahora guardamos y cerramos. Reinciamos nuestro Nagios para que se apliquen los cambios:

{% highlight bash %}
sudo service nagios reload
{% endhighlight %}

Una vez a configurado Nagios en todos los equipos o host que desee monitorizar todo habrá acabado. Asegurese de tener acceso a su interface web Nagios y chequear la página de servicios para ver todos los equipos y servicios monitorizados.