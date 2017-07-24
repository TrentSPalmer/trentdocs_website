# Misc Tips, TroubleShooting

## Sending commands to LXD containers

Use `bash -c "<command>"` for commands with wildcards. i.e.

```bash
for machine in $(lxc list | grep RUNNING | awk '{print $2}') ;\
    do lxc exec "${machine}" -- bash -c "cat /etc/apt/apt.conf.d/02*" ; done
```

## Ubuntu-Mate-Welcome-Center doesn't work some for some repos

Perhaps your apt-cacher-ng proxy server isn't configured to allow 
traffic through from https sources. Make sure the following is
uncommented. This applies for all PPA's that use https.

```conf
# /etc/apt-cacher-ng/acng.conf
PassThroughPattern: .*
```
