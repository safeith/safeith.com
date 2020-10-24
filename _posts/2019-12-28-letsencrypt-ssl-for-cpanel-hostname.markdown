---
layout: post
title:  "Let's Encrypt SSL for cPanel hostname"
date:   2019-12-28 11:38:00 +0100
categories: SSL
---
In this short story, I will share my experience about using the Let's Encrypt SSL certificate for cPanel hostname. As you know, cPanel provides a free SSL certificate for cPanel hostname as default. But some times it now works for Iranian domain, so you can use the following steps to have a valid SSL certificate for your cPanel services.

## Install the `Certbot` from `EPEL`
Run following command to install `certbot` from `epel` repo
{% highlight bash %}
yum install --enablerepo=epel certbot
{% endhighlight %}

## Create `deploy-hook` script for `Certbot`
Create `hostname-ssl.sh` file
{% highlight bash %}
vim /usr/local/bin/hostname-ssl.sh
# Copy following lines on it
#!/bin/sh
set -e

/bin/cat /etc/letsencrypt/live/$HOSTNAME/privkey.pem /etc/letsencrypt/live/$HOSTNAME/cert.pem > /var/cpanel/ssl/cpanel/cpanel.pem
/bin/chown cpanel:cpanel /var/cpanel/ssl/cpanel/cpanel.pem

/bin/cat /etc/letsencrypt/live/$HOSTNAME/privkey.pem > /var/cpanel/ssl/exim/exim.key
/bin/cat /etc/letsencrypt/live/$HOSTNAME/cert.pem > /var/cpanel/ssl/exim/exim.crt
/bin/chown mailnull:mail /var/cpanel/ssl/exim/exim.*

/bin/cat /etc/letsencrypt/live/$HOSTNAME/privkey.pem > /var/cpanel/ssl/ftp/ftpd-rsa-key.pem
/bin/cat /etc/letsencrypt/live/$HOSTNAME/cert.pem > /var/cpanel/ssl/ftp/ftpd-rsa.pem
/bin/cat /etc/letsencrypt/live/$HOSTNAME/privkey.pem /etc/letsencrypt/live/$HOSTNAME/cert.pem > /var/cpanel/ssl/ftp/pure-ftpd.pem
/bin/chown root:wheel /var/cpanel/ssl/ftp/*

/bin/cat /etc/letsencrypt/live/$HOSTNAME/privkey.pem > /var/cpanel/ssl/dovecot/dovecot.key
/bin/cat /etc/letsencrypt/live/$HOSTNAME/cert.pem > /var/cpanel/ssl/dovecot/dovecot.crt
/bin/chown root:wheel /var/cpanel/ssl/dovecot/dovecot.*

/bin/systemctl restart cpanel.service
/bin/systemctl restart exim.service
/bin/systemctl restart pure-ftpd.service
/bin/systemctl restart dovecot.service
{% endhighlight %}

Now make it executable
{% highlight bash %}
chmod +x /usr/local/bin/hostname-ssl.sh
{% endhighlight %}

## Issue a certificate for cPanel hostname
With the following command you will be able to issue a Let's Encrypt valid certificate for cPanel HOSTNAME
{% highlight bash%}
certbot --debug certonly -a webroot --agree-tos --webroot-path=/usr/local/apache/htdocs --deploy-hook=/usr/local/bin/hostname-ssl.sh --renew-by-default -d $HOSTNAME
{% endhighlight %}

## Certificate renew cron job
For the certificate, auto-renew add the following cron job
{% highlight bash %}
00 02 * * * certbot renew
{% endhighlight %}

---
Source: [cPanel Forums](https://forums.cpanel.net/threads/cpanel-ssl-certs-lets-encrypt.522041/)
