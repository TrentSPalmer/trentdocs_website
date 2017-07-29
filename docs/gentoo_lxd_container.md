# Gentoo LXD Container

## There are Gentoo images at [linuxcontainers.org](https://us.images.linuxcontainers.org/)

```bash
lxc image list images: | grep gentoo
lxc init images:34760012759f
# or
lxc init images:34760012759f <pick a name>
```

## Networking

The default image will request dhcp service on eth0. If you need a second static
connection on eth1, do the following. Describe eth1 in `/etc/conf.d/net`

```conf
# /etc/conf.d/net
config_eth1="10.44.84.101 netmask 255.255.255.0 brd 10.44.84.255"
routes_eth1="default via 10.44.84.1"
```

Then in `/etc/init.d/`
```bash
ln -s /etc/init.d/net.lo /etc/init.d/net.eth1
```

Enable net.eth1 in init.
```bash
rc-update add net.eth1 default
```

And then start networking on eth1
```bash
/etc/init.d/net.eth1 start
```

## Locale and Timezone

You're supposed to write your timezone in `/etc/timezone`, `echo "Europe/Brussels" > /etc/timezone`,
and then run the command `emerge --config sys-libs/timezone-data`. But this doesn't work.

You can set the locale by uncommenting your locale in `/etc/locale-gen`, and then
running the following commands.

```bash
locale-gen
eselect locale list
eselect locale set <number>
. /etc/profile
```

And the following corrected the timezone.

```bash
unlink /etc/localtime
ln -s /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
```

