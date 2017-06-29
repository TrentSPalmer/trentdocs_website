# Quick Dirty Redis Nspawn Container on Arch Linux

Refer to the [Nspawn](nspawn.md) page for setting up the nspawn container,
install redis, and start/enable redis.service.
Once you have the container running, it seems all you have to do to get
things working in a container subnet is to change the bind address.

```text
# /etc/redis.conf
# bind 127.0.0.1
bind 0.0.0.0
```

you can nmap port 6379, be sure to restart redis

[Again I would refer you to the Arch Wiki](https://wiki.archlinux.org/index.php/Redis)
