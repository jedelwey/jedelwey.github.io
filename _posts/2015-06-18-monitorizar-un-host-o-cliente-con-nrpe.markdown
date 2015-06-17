---
layout: post
title:  "Monitorizar un Host o Cliente con NRPE"
date:   2015-06-18 00:26:00
categories: nagios
---

#Monitor a Host with NRPE
In this section, we'll show you how to add a new host to Nagios, so it will be monitored. Repeat this section for each server you wish to monitor.

On a server that you want to monitor, update apt-get:

{% highlight bash %}
sudo apt-get update
{% endhighlight %}

Now install Nagios Plugins and NRPE:

{% highlight bash %}
sudo apt-get install nagios-plugins nagios-nrpe-server
{% endhighlight %}

Now, let's update the NRPE configuration file. Open it in your favorite editor (we're using vi):

{% highlight bash %}
sudo nano /etc/nagios/nrpe.cfg
{% endhighlight %}

Find the allowed_hosts directive, and add the private IP address of your Nagios server to the comma-delimited list (substitute it in place of the highlighted example):

{% highlight bash %}
allowed_hosts=127.0.0.1,10.132.224.168
{% endhighlight %}

Save and exit. This configures NRPE to accept requests from your Nagios server, via its private IP address.

Restart NRPE to put the change into effect:

{% highlight bash %}
sudo service nagios-nrpe-server restart
{% endhighlight %}

Once you are done installing and configuring NRPE on the hosts that you want to monitor, you will have to add these hosts to your Nagios server configuration before it will start monitoring them.

#Add Host to Nagios Configuration

On your Nagios server, create a new configuration file for each of the remote hosts that you want to monitor in /usr/local/nagios/etc/servers/. Replace the highlighted word, "yourhost", with the name of your host:

{% highlight bash %}
sudo nano /usr/local/nagios/etc/servers/yourhost.cfg
{% endhighlight %}

Add in the following host definition, replacing the host_name value with your remote hostname ("web-1" in the example), the alias value with a description of the host, and the address value with the private IP address of the remote host:

{% highlight bash %}
define host {
        use                             linux-server
        host_name                       yourhost
        alias                           My first Apache server
        address                         10.132.234.52
        max_check_attempts              5
        check_period                    24x7
        notification_interval           30
        notification_period             24x7
}
{% endhighlight %}

With the configuration file above, Nagios will only monitor if the host is up or down. If this is sufficient for you, save and exit then restart Nagios. If you want to monitor particular services, read on.

Add any of these service blocks for services you want to monitor. Note that the value of check_command determines what will be monitored, including status threshold values. Here are some examples that you can add to your host's configuration file:

Ping:

{% highlight bash %}
define service {
        use                             generic-service
        host_name                       yourhost
        service_description             PING
        check_command                   check_ping!100.0,20%!500.0,60%
}
{% endhighlight %}

SSH (notifications_enabled set to 0 disables notifications for a service):

{% highlight bash %}
define service {
        use                             generic-service
        host_name                       yourhost
        service_description             SSH
        check_command                   check_ssh
        notifications_enabled           0
}
{% endhighlight %}

If you're not sure what use generic-service means, it is simply inheriting the values of a service template called "generic-service" that is defined by default.

Now save and quit. Reload your Nagios configuration to put any changes into effect:

{% highlight bash %}
sudo service nagios reload
{% endhighlight %}

Once you are done configuring Nagios to monitor all of your remote hosts, you should be set. Be sure to access your Nagios web interface, and check out the Services page to see all of your monitored hosts and services: