# Dynamic Cacheing Nginx Reverse Proxy For Pacman

## You set up a dynamic cacheing reverse proxy and then you put the ip address or hostname for that server in `/etc/pacman.d/mirrorlist` on your client machines.

Of course if you want to you can set this up and run it in an
[Nspawn Container](nspawn.md).
The [ArchWiki Page for pacman tips](https://wiki.archlinux.org/index.php/Pacman/Tips_and_tricks#Dynamic_reverse_proxy_cache_using_nginx)
mostly spells out what to do, but I want to document
the exact steps I would take.

As for how you would run this on a server with other virtual hosts?
Who cares? That is what is so brilliant about using using an
nspawn container, in that it behaves like just another
computer on the lan with it's own ip address.  But it only does one
thing, and that's all you have to configure it for.

I see no reason to use nginx-mainline instead of stable.

```bash
pacman -S nginx
```

The suggested configuration in the Arch Wiki
is to create a directory `/srv/http/pacman-cache`,
and that seems to work well enough

```bash
mkdir /srv/http/pacman-cache
# and then change it's ownershipt
chown http:http /srv/http/pacman-cache
```

## nginx configuration

and then it references an nginx.conf in
[this gist](https://gist.github.com/anonymous/97ec4148f643de925e433bed3dc7ee7d),
but that is not a complete nginx.conf and so here is a method to get that
working as of July 2017 with a fresh install of nginx.

You can start with a default `/etc/nginx/nginx.conf`,
and add the line `include sites-enabled/*;`
at the end of the *http* section.

```text
# /etc/nginx/nginx.conf
#user html;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}
    include sites-enabled/*;

}
```

And then create the directory `/etc/nginx/sites-enabled`

```bash
mkdir /etc/nginx/sites-enabled
```

And then create `/etc/nginx/sites-enabled/proxy_cache.conf`,
which is *mostly* a
[copy-and-paste from this gist](https://gist.github.com/anonymous/97ec4148f643de925e433bed3dc7ee7d).

Notice the *server_name*. This has to match the entry in
`/etc/pacman.d/mirrorlist` on the client machines you are
updating from. If you can use the hostname, great. But if you
have to assign static ip addresses and explicitly write the local
ip address instead, then that should match what you write in your mirrorlist.

And of course your mirrorlist entry
on the client machine, has to preserve the directory scheme.

```text
# /etc/pacman.d/mirrorlist
Server = http://<hostname or ip address>:<port if not 80>/archlinux/$repo/os/$arch
```

```text
# /etc/nginx/sites-enabled/proxy_cache.conf
# nginx may need to resolve domain names at run time
resolver 8.8.8.8 8.8.4.4;

# Pacman Cache
server
{
listen      80;
server_name <hostname or ip address>; # has to match the entry in mirrorlist on client machine.
root        /srv/http/pacman-cache;
autoindex   on;

	# Requests for package db and signature files should redirect upstream without caching
	# Well that's the default anyway.
	# But what if you're spinning up a lot of nspawn containers, don't want to waste all that bandwidth?
	# I choose to instead run a systemd timer that deletes the *db files once every 15 minutes
	location ~ \.(db|sig)$ {
	    try_files $uri @pkg_mirror;
	    # proxy_pass http://mirrors$request_uri;
	}

	# Requests for actual packages should be served directly from cache if available.
	#   If not available, retrieve and save the package from an upstream mirror.
	location ~ \.tar\.xz$ {
	    try_files $uri @pkg_mirror;
	}

	# Retrieve package from upstream mirrors and cache for future requests
	location @pkg_mirror {
	    proxy_store    on;
	    proxy_redirect off;
	    proxy_store_access  user:rw group:rw all:r;
	    proxy_next_upstream error timeout http_404;
	    proxy_pass          http://mirrors$request_uri;
	}
}

# Upstream Arch Linux Mirrors
# - Configure as many backend mirrors as you want in the blocks below
# - Servers are used in a round-robin fashion by nginx
# - Add "backup" if you want to only use the mirror upon failure of the other mirrors
# - Separate "server" configurations are required for each upstream mirror so we can set the "Host" header appropriately
upstream mirrors {
server localhost:8001;
server localhost:8002; # backup
server localhost:8003; # backup
}

# Arch Mirror 1 Proxy Configuration
server
{
listen      8001;
server_name localhost;

	location / {
	    proxy_pass       http://mirrors.kernel.org$request_uri;
	    proxy_set_header Host mirrors.kernel.org;
	}
}

# Arch Mirror 2 Proxy Configuration
server
{
listen      8002;
server_name localhost;

	location / {
	    proxy_pass       http://mirrors.ocf.berkeley.edu$request_uri;
	    proxy_set_header Host mirrors.ocf.berkeley.edu;
	}
}

# Arch Mirror 3 Proxy Configuration
server
{
	listen      8003;
	server_name localhost;

	location / {
	    proxy_pass       http://mirrors.cat.pdx.edu$request_uri;
	    proxy_set_header Host mirrors.cat.pdx.edu;
	}
}
```

## systemd service that cleans the proxy cache

### don't enable the service, enable the timer

```bash
systemctl enable/start /etc/systemd/system/proxy_cache_clean.timer
```

Keeps the 2 most recent versions of each package using paccache command.

```text
# /etc/systemd/system/proxy_cache_clean.service
[Unit]
Description=Clean The pacman proxy cache

[Service]
Type=oneshot
ExecStart=/usr/bin/find /srv/http/pacman-cache/ -type d -exec /usr/bin/paccache -v -r -k 2 -c {} \;
StandardOutput=syslog
StandardError=syslog
```

## systemd timer for the systemd service that cleans the proxy cache

```text
# /etc/systemd/system/proxy_cache_clean.timer
[Unit]
Description=Timer for clean The pacman proxy cache

[Timer]
OnBootSec=20min
OnUnitActiveSec=100h
Unit=proxy_cache_clean.service

[Install]
WantedBy=timers.target
```

## systemd service that deletes the pacman database files from the proxy cache

### don't enable the service, enable the timer

```bash
systemctl enable/start /etc/systemd/system/proxy_cache_database_clean.timer
```

You won't need this if you don't cache the database files. But if you do cache
the database files, then you'll just be stuck with old database files, unless
you periodically delete them. But I'm not sure about all this, will keep an
eye on things.

```text
# /etc/systemd/system/proxy_cache_database_clean.service
[Unit]
Description=Clean The pacman proxy cache database

[Service]
Type=oneshot
ExecStart=/bin/bash -c "for f in $(find /srv -name *db) ; do rm $f; done"
StandardOutput=syslog
StandardError=syslog
```

## systemd timer for the systemd service that deletes the pacman database files from the proxy cache

```text
# /etc/systemd/system/proxy_cache_database_clean.timer
[Unit]
Description=Timer for clean The pacman proxy cache database

[Timer]
OnBootSec=10min
OnUnitActiveSec=15min
Unit=proxy_cache_database_clean.service

[Install]
WantedBy=timers.target
```
