---
layout: post
title:  "Monitorizar un Host o Cliente Windows con NSClient++"
date:   2015-06-18 01:26:00
categories: nagios
---

# Monitorizar un Host o cliente Windows con NSClient++

En este post, expplicaré como añadir un host o cliente Windows a nuestro nagios para ser monitorizado. Solo tienes que repetir este post para cada equipo que se quiera monitorizar. 

Lo primero que tenemos que hacer es descargarnos la [última versión de NSClient++][NSClient++] de su página web. Una vez descargado el programa, comenzaremos su instalación:

Seleccionamos Ejecutar:

<img src="https://davidduranmartos.github.io/images/instalador.PNG"/>

Seleccionamos Next:

<img src="https://davidduranmartos.github.io/images/siguiente.PNG"/>

Aceptamos los terminos y le damos a next:

<img src="https://davidduranmartos.github.io/images/acepto-siguiente.PNG"/>

Seleccionamos la instalación Typical y le damos a next:

<img src="https://davidduranmartos.github.io/images/no-tocar-siguiente.PNG"/>

Aquí es muy importante hacer algunos cambios:
    - Añadir la dirección IP del servidor Nagios en ``Allowed_host``
    - ``Enable common check plugins para tener los check más comunes
    - ``Enable nsclient server (check_nt)`` para poder hacer checks de windows como el uptime, la cpu, procesos, memoria, etc.
    - ``Enable NRPE server (check_nrpe)`` y seleccionar la opción ``Insecure legacy mde (required by old check_nrpe)``
    
<img src="https://davidduranmartos.github.io/images/crear-esta-configuracion.PNG"/>

Y procedemos a darle a Install para que se lleve a cabo la instalación con los parametros acordados.

<img src="https://davidduranmartos.github.io/images/install.PNG"/>

Una vez acabada la instalación solo tendremos que darle a Finish y se cerrará el instalador.

<img src="https://davidduranmartos.github.io/images/finish.PNG"/>

Con esto ya tendremos instalado el cliente, ahora hay que hacer una pequeña modificación de servicio ``NSClient++ (x64)`` para  que al iniciar sesión actute con una cuenta del sistema y ``Permitir que el servicio interactúe con el escritorio``

Para ello abriremos servicios, una manera de abrirlo es pulsar el botón de inicio y escribir ``servicios`` como se puede ver en la imagen sale como primera opcion:

<img src="https://davidduranmartos.github.io/images/servicios.PNG"/>

Buscamos dentros de los servicios el servicio ``NSClient++ (x64)`` (puede variar si el equipo es x86) y lo abrimos. Nos vamos a la pestaña de Iniciar sesión y dejamos marcado la opción: ``Permitir que el servicio interactúe con el escritorio``

<img src="https://davidduranmartos.github.io/images/permitir.PNG"/>

Aplicamos, aceptamos y cerramos todo. Ya tenemos todo instalado en nuestro cliente, ahora falta añadir el host en el servidor.

# Añadir un Host o cliente a nuestra configuración de Nagios
Editamos el fichero ``nagios.cfg`` que está en /usr/local/nagios/etc/nagios.cfg y descomentamos el ``windows.cfg`` quitando el # de la primera línea:

{% highlight bash %}
cfg_file=/usr/local/nagios/etc/objects/windows.cfg
{% endhighlight %}

Ahora procederemos a editar el fichero ``windows.cfg`` en el cual buscaremos y sustituiremos ``winserver`` por el nombre de nuestro host o equipo, y cambiaremos también su address y su alias.

{% highlight bash %}
define host{
        use             windows-server  ; Inherit default values from a template
        host_name       W7-Oficina   ; The name we're giving to this host
        alias           Servidor W7 de intercambio de ficheros       ; A longer name associated with the host
        address         192.168.1.52    ; IP address of the host
        }
{% endhighlight %}

Recordar que tenemos que hacer esto tambien en el grupo si queremos renombrarlo y sobre todo en los servicios:

{% highlight bash %}
define service{
        use                     generic-service
        host_name               W7-Oficina
        service_description     NSClient++ Version
        check_command           check_nt!CLIENTVERSION
        }
{% endhighlight %}

Posteriormente reinciamos Nagios y comprobamos que todo funciona perfectamente

{% highlight bash %}
sudo service nagios restart
{% endhighlight %}

[NSClient++]: http://www.nsclient.org/download/