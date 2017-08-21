# Misc Tips, TroubleShooting

## Sending commands to LXD containers

Use `bash -c "<command>"` for commands with wildcards. i.e.

```bash
for machine in $(lxc list | grep RUNNING | awk '{print $2}') ;\
    do lxc exec "${machine}" -- bash -c "cat /etc/apt/apt.conf.d/02*" ; done
```

fish shell is actually a little bit cleaner

```fish
for machine in (lxc list | grep RUNNING | awk '{print $2}') ; \
    lxc exec $machine -- bash -c "cat /etc/apt/apt.conf.d/02*" ; end
```

```fish
# change all their time zones
for machine in (lxc list | grep RUNNING | awk '{print $2}') ; \
    lxc exec $machine -- bash -c "timedatectl set-timezone America/Los_Angeles" ; end
# check to see if anyone is logged in before rebooting
for machine in (lxc list | grep RUNNING | awk '{print $2}') ; echo ; \
    echo $machine ; lxc exec $machine -- bash -c "who" ; end 
```

## Move LXD container to another Server

```bash
# stop the container
lxc stop <container name>
# publish image of container to local *storage*
lxc publish <container name> --alias <image name>
# export the new image to tarball
lxc image export <image name>
# scp tarball to other box
scp a4762b114fecee2e2bc227b9032405642c5286c02009babef7953e011e597bfe.tar.gz server:
# on other box import the image
lxc image import <tarball file name> --alias <image name>
# launch container
lxc launch <image name> <container name>
# assign profile to container
lxc profile assign <container name> <profile name>
```

Shell into the new running container, update any network interface
configurations that you need to, and then restart the container.

See also
[LXD Container Home Server Networking For Dummies](lxd_container_home_server_networking_for_dummies.md)

## Ubuntu-Mate-Welcome-Center doesn't work some for some repos

Perhaps your apt-cacher-ng proxy server isn't configured to allow 
traffic through from https sources. Make sure the following is
uncommented. This applies for all PPA's that use https.

```conf
# /etc/apt-cacher-ng/acng.conf
PassThroughPattern: .*
```

## Quitting Mosh

The key combination to quit mosh it `ctrl+6+.`
Also, WTF?

## Updating Caddy Server

You update Caddy Server with a new Go Binary, try to restart caddy.service, and it fails.
Maybe you get an error message such as the following `listen tcp :80: bind: permission denied` and/or
`listen tcp :443: bind: permission denied`.  
Fix this error with the following command `sudo setcap CAP_NET_BIND_SERVICE=+eip /path/to/caddy`
