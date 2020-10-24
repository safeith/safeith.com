---
layout: post
title:  "SSH over Tor hidden service"
date:   2019-03-15 22:19:32 +0430
categories: Security
---
Sometimes you need to access your device with ssh connection, but you don't have a static public IP on your machine, or you're using a shared internet connection and want to give ssh access to somebody to connect to your device. What can you do? Maybe you know some ways, but I prefer to use `Tor hidden` service. In this small article, I will write how I do it on Archlinux.

In the first step, I install `ssh`, `tor`, and `torsocks`

{% highlight bash %}
sudo pacman -S ssh tor torsocks
{% endhighlight %}

In the second step, I configure tor to enable hidden service for ssh

{% highlight bash %}
sudo echo 'HiddenServiceDir /var/lib/tor/ssh_hidden_service/' >> /etc/tor/torrc
sudo echo 'HiddenServicePort 22 127.0.0.1:22' >> /etc/tor/torrc
{% endhighlight %}

In the third step, I start ssh and tor service 
{% highlight bash %}
sudo systemctl start sshd.service tor.service
{% endhighlight %}

In the fourth step, I check hostname of hidden service 
{% highlight bash %}
sudo cat /var/lib/tor/ssh_hidden_service/hostname
{% endhighlight %}

In the fifth step, I check if it works ok
{% highlight bash %}
torify ssh hidden_hostname
{% endhighlight %}

