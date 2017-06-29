# Nspawn Containers

[This Link For Arch Linux Wiki for Nspawn Containers](https://wiki.archlinux.org/index.php/Systemd-nspawn)

I like the idea of starting with the easy containers first.

### Create a FileSystem

```bash
cd /var/lib/machines
# create a directory
mkdir <container>
# use pacstrap to create a file system
pacstrap -i -c -d <container> base --ignore linux
```


At this point you might want to copy over some configs to save time later.

* /etc/locale.conf
* /root/.bashrc
* /etc/locale.gen

### First boot and create root password

```bash
systemd-nspawn -b -D <container>
passwd
# assuming you copied over /etc/locale.gen
locale-gen
# set timezone
timedatectl set-timezone <timezone>
# enable network time
timedatectl set-ntp 1
# enable networking
systemctl enable systemd-networkd
systemctl enable systemd-resolved
poweroff
# if you want to nat the container add *-n* flag
systemd-nspawn -b -D <container> -n
# and to bind mount the package cache
systemd-nspawn -b -D <container> -n --bind=/var/cache/pacman/pkg
```

### Networking
Here's a link that skips ahead to [Automatically Starting the Container](#automatically-starting-the-container)

On Arch, assuming you have systemd-networkd and systemd-resolved
set up correctly, networking from the host end of things should
just work.  
However on Linode it does not. What does work on Linode is to create
a bridge interface. Two files for br0 will get the job done.

```text
# /etc/systemd/network/50-br0.netdev
[NetDev]
Name=br0
Kind=bridge
```

```text
# /etc/systemd/network/50-br0.netdev
[Match]
Name=br0

[Network]
Address=10.0.55.1/24 # arbitrarily pick a subnet range to taste
DHCPServer=yes
IPMasquerade=yes
```

Notice how the configuration file tells systemd-networkd to offer
DHCP service and to perform masquerade. You can modify the `systemd-nspawn`
command to use the bridge interface. Every container attached to this bridge
will be on the same subnet and able to talk to each other.

```bash
# first restart systemd-networkd to bring up the new bridge interface
systemctl restart systemd-networkd
# and add --network-bridge=br0 to systemd-nspawn command
systemd-nspawn -b -D <container> --network-bridge=br0 --bind=/var/cache/pacman/pkg
```

### Automatically Starting the Container
Here's a link back up to [Networking](#networking)
in case you previously skipped ahead.

There are two ways to automate starting the container. You can override
`systemd-nspawn@.service` or create an *nspawn* file.  

First enable machines.target

```bash
# to override the systemd-nspawn@.service file
cp /lib/systemd/system/systemd-nspawn@.service /etc/systemd/system/systemd-nspawn@<container>.service
```
Edit `/etc/systemd/system/systemd-nspawn@<container>.service` to add the `systemd-nspawn` options
you want to the `ExecStart` command.

Or create `/etc/systemd/nspawn/<container>.nspawn`

```text
# /etc/systemd/nspawn/<container>.nspawn
[Files]
Bind=/var/cache/pacman/pkg

[Network]
Bridge=br0
```

```text
# /etc/systemd/nspawn/<container>.nspawn
[Files]
Bind=/var/cache/pacman/pkg

[Network]
VirtualEthernet=1 # this seems to be the default sometimes, though
```

```bash
# in either case
systemctl start/enable systemd-nspawn@<container>
# to get a shell
machinectl shell <container>
# and then to get an environment
bash
```

This would be a good time to check for network and name resolution,
symlink resolv.conf if need be.

### Initial Configuration Inside The Container

```bash
# set time zone if you don't want UTC
timedatectl set-timezone <timezone>
# enable ntp, networktime
timedatectl set-ntp 1
# enable networking from inside the container
systemctl enable systemd-networkd
systemctl start systemd-networkd
systemctl enable systemd-resolved
systemctl start systemd-resolved
rm /etc/resolv.conf 
ln -s /run/systemd/resolve/resolv.conf /etc/
# ping google
ping -c 3 google.com
```

[If you want to change the locale](https://wiki.archlinux.org/index.php/locale)

## Final Observations
* You can start/stop nspawn containers with `machinectl` command. 
* You can start nspawn containers with `systemd-nspawn` command.
* You can configure the systemd service for a container with @nspawn.service file override
* Or you can configure an nspawn container with a dot.nspawn file

But in regards to the above list
I have noticed differences in behaviour,
in some scenarios, concerning file attributes
for bind mounts.

Another curiosity: when you have nspawn containers natted on VirtualEthernet connections,
they might be able to ping each other at 10.x.y.z, but not resolve each other. But they might
be able to resolve each other if they are all connected to the same bridge interface or nspawn
network zone, but will randomly resolve each other in any of the 10.x.y.z, 169.x.y.z,
or fe80::....:....:....%host (ipv6 local) spaces, which would complicate configuring the containers
to talk to each other. But I intend to look into this some more.
