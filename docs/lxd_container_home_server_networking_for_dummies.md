# LXD Container Home Server Networking For Dummies

## Why?
If you're going to operate a fleet of LXD containers for home
entertainment, you probably want some of them exposed with their
own ip addresses on your home network, so that you can use them
as containerized servers for various applications.

Others containers, you might want to be inaccessable from the lan,
in a natted subnet, where they can solicit connections to the
outside world from within their natted subnet, but are not addressable
from the outside. A database server that you connect a web app to, for
instance, or a web app that you have a reverse proxy in front of.

But these are two separate address spaces, so ideally all of the containers
would have a second interface of their own, by which they could connect
to a third network, that would be a private network that all of the containers
can use to talk directly to each other (or the host machine).

It's pretty straightforward, you just have to glue all the pieces together.

## Three Part Overview.

1. Define and create some bridges.  

2. Define profiles that combine the network
interfaces in different combinations. In addition to two
bridges you will have a macvlan with which to expose the containers
that you want exposed, but the macvlan doesn't come into
play until here in step two when you define profiles.  

3. Assign each container which profile it should use,
and then configure the containers to use the included
network interfaces correctly.  

## Build Sum Moar Bridges

The containers will all have two network interfaces from
their own internal point of view, *eth0* and *eth1*. 

In this
scheme we create a bridge for a natted subnet and a bridge for
a non-natted subnet. All of the containers will connect to the
non-natted subnet on their second interface, *eth1*, and some
of the containers will connect to the natted subnet on their 
first interface *eth0*. The containers that don't connect
to the natted subnet will instead connect to a macvlan
on their first interface *eth0*, but that isn't part of this
step.

### bridge for a natted subnet

If you haven't used lxd before, you'll want to run the command `lxd init`.
By default this creates exactly the bridge we want, called *lxdbr0*.

Otherwise you would use the following command to create *lxdbr0*.

```bash
lxc network create lxdbr0
```

To generate a table of all the existing interfaces.

```bash
lxd network list
```

This bridge is for our natted subnet, so we just want to go with
the default configuration.

```bash
lxc network show lxdbr0
```

This cats a yaml file where you can see the randomly
generated network for *lxdbr0*.

```yaml
config:
  ipv4.address: 10.99.153.1/24
  ipv4.nat: "true"
  ipv6.address: fd42:211e:e008:954b::1/64
  ipv6.nat: "true"
description: ""
name: lxdbr0
type: bridge
used_by: []
managed: true
```

### bridge for a non-natted subnet

Create *lxdbr1*

```bash
lxc network create lxdbr1
```

Use the following commands to remove nat from 
lxdbr1.

```bash
lxc network set lxdbr1 ipv4.nat false
lxc network set lxdbr1 ipv6.nat false
```

Of if you use this next command, your favourite
text editor will pop open, preloaded with the complete yaml file
and you can edit the configuration there.

```bash
lxc network edit lxdbr1
```

Either way you're looking for a result such as the following.
Notice that the randomly generated address space is different
that the one for *lxdbr0*, and that the *nat keys are set
to "false".

```yaml
config:
  ipv4.address: 10.151.18.1/24
  ipv4.nat: "false"
  ipv6.address: fd42:89d4:f465:1b20::1/64
  ipv6.nat: "false"
description: ""
name: lxdbr1
type: bridge
used_by: []
managed: true
```

## Profiles

### recycle the default
When you first ran `lxd init`, that created a default profile.
Confirm with the following.

```bash
lxc profile list
```

To see what the default profile looks like.

```bash
lxc profile show default
```

```yaml
config:
  environment.http_proxy: ""
  security.privileged: "true"
  user.network_mode: ""
description: Default LXD profile
devices:
  eth0:
    nictype: bridged
    parent: lxdbr0
    type: nic
  root:
    path: /
    pool: default
    type: disk
name: default
used_by: []
```

### profile the natted

The easiest way to create a new profile is start by copying another one.

```bash
lxc profile copy default natted
```

edit the new *natted* profile

```bash
lxc profile edit natted
```

And add an *eth1* interface attached to *lxdbr1*. *eth0* and *eth1* will
be the interfaces visible from the container's point of view.

```yaml
config:
  environment.http_proxy: ""
  security.privileged: "true"
  user.network_mode: ""
description: Natted LXD profile
devices:
  eth0:
    nictype: bridged
    parent: lxdbr0
    type: nic
  eth1:
    nictype: bridged
    parent: lxdbr1
    type: nic
  root:
    path: /
    pool: default
    type: disk
name: natted
used_by: []
```

Any container assigned to the *natted* profile, will have an interface *eth0* connected
to a natted subnet, and a second interface *eth1* connected to a non-natted subnet, with
a static ip on which it will be able to talk directly to the other containers and the host
machine.

### profile the exposed

Create the *exposed* profile

```bash
lxc profile copy natted exposed
```

and edit the new *exposed* profile

```bash
lxc profile edit exposed
```

change the nictype for *eth0* from `bridged` to `macvlan`, and the parent should be
the name of the physical ethernet connection on the host machine, instead of a bridge.

```yaml
config:
  environment.http_proxy: ""
  security.privileged: "true"
  user.network_mode: ""
description: Exposed LXD profile
devices:
  eth0:
    nictype: macvlan
    parent: eno1
    type: nic
  eth1:
    nictype: bridged
    parent: lxdbr1
    type: nic
  root:
    path: /
    pool: default
    type: disk
name: exposed
used_by: []
```

Any container assigned to the *exposed* profile, will have an interface *eth0* connected
to a macvlan, addressable from your lan, just like any other arbitrary computer on
your home network, and a second interface *eth1* connected to a non-natted subnet, with
a static ip on which it will be able to talk directly to the other containers and the host
machine.

## Assign Containers to Profiles and configure them to connect correctly.

There are a lot of different ways that a Linux instance can solicit network services. So for
now I will just describe a method that will work here for a lxc container from ubuntu:16.04, as
well as a debian stretch container from images.linuxcontainers.org.

Start a new container and assign the profile. We'll use an arbitrary whimsical container name,
*quick-joey*. This process is the same for either the *natted* profile or the *exposed* profile.

```bash
lxc init ubuntu:16.04 quick-joey
# assign the profile
lxc profile assign quick-joey exposed
# start quick-joey
lxc start quick-joey
# and start a bash shell
lxc exec quick-joey bash
```

You need to tell these containers how to connect to the non-natted subnet on *eth1*
With either an ubuntu:16.04 container, or a debian stretch container, for either the *natted* or
*exposed* profile, because of all the above configuration work they will automatically connect on
their *eth0* interfaces and be able to talk to the internet. You need to edit `/etc/network/interfaces`,
the main difference being what that file looks like before you edit it.

### ubuntu:16.04

If you start a shell on an ubuntu:16.04 container, you see that `/etc/network/interfaces`
describes the loopback device for localhost, then sources `/etc/network/interfaces.d/*.cfg` where
some magical cloud-config jazz is going on. You just want to add a static ip description for *eth1*
to the file `/etc/network/interfaces`. And obviously take that the static ip address you assign is
unique and on the same subnet with *lxdbr1*.

Reminder: the address for *lxdbr1* is 10.151.18.1/24, but it will be different on your machine.

```conf
auto lo
iface lo inet loopback

source /etc/network/interfaces.d/*.cfg
# what you add goes below here
auto eth1
iface eth1 inet static
   address 10.151.18.123
   netmask 255.255.255.0
   broadcast 255.255.255.255 
   network 10.151.18.0
```

### debian stretch

The configuration for a debian stretch container is the same, except the the file
`/etc/network/interfaces` will also describe eth0, but you only have to add the 
description for eth1.

### the /etc/hosts file

Once you assign the containers static ip addresses for their *eth1*
interfaces, you can use the `/etc/hosts` file on each container to make them
aware of where the other containers and the host machine are.

For instance, if you want the container *quick-joey* to talk directly
to the host machine, which will be at the ip address of *lxdbr1*, start a shell
on the container *quick-joey*

```bash
lxc exec quick-joey bash
```

and edit `/etc/hosts`

```conf
# /etc/hosts
10.151.18.1    mothership
```

Of you have a container named *fat-cinderella*, that needs to be able to talk
directly *quick-joey*.

```bash
lxc exec fat-cinderella bash
vim /etc/hosts
```

```conf
# /etc/hosts
10.151.18.123  quick-joey
```

etcetera



