# Selfoss on Centos 7

The target here is a very low resource vps running Centos7.
You can use mysql or postgresql, but performance is fine with sqlite database.
You'll need the epel repo in order to install python2-certbot-apache.

[Here's a great guide for setting up apache with letsencrypt on Centos7](https://www.digitalocean.com/community/tutorials/how-to-secure-apache-with-let-s-encrypt-on-centos-7).

You'll want to install the following packages

* mod_ssl
* python2-certbot-apache
* php
* php-gd
* php-http
* php-pdo
* unzip
* wget

[The documentation](https://selfoss.aditu.de/) explains how to set up
the config.ini and .htaccess files, RewriteEngine, RewriteBase,
database, and explains the apache modules that you
want enabled. Hint, use `apachectl -M`, `apachectl help`, etc.

You'll probably want to extract the 
application to `/var/www/html/selfoss/` or similar, and then add a configuration.

```conf
# /etc/httpd/conf.d/selfoss.conf
Alias "/selfoss/" "/var/www/html/selfoss/"
<Directory "/var/www/html/selfoss">
        Options FollowSymLinks
        AllowOverride All
</Directory>
```

Make sure that the selfoss directory is owned by apache:apache.
