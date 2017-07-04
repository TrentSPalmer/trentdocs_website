# Debian Nspawn Container On Arch For Testing Apache Configurations

Begin by exporting the environmental variable for your squid cacheing 
proxy. If you're deboostrapping Debian file systems, the best way to
speed this up is with squid.

The ArchWiki page for nspawn containers has a
[Debian/Ubuntu subsection](https://wiki.archlinux.org/index.php/Systemd-nspawn#Create_a_Debian_or_Ubuntu_environment)
Obviously you're going to want to install debootstrap and debian-archive-keyring.

```bash
# to create a Stretch Container
cd /var/lib/machines 
mkdir <container name> 
deboostrap stretch <container name>
```

After some experimentation, perhaps this is the best time to write
the intended hostname into the container, and write any
apt-cacher or apt-cacher-ng proxies into /etc/apt/apt.conf 
on the container.

```bash
cp apt.conf /etc/apt/apt.conf 
echo "<hostname>" > /var/lib/machines/<container name>/etc/hostname
```

And then start the container, and set the root password.

```bash 
# boot in interactive mode
systemd-nspawn -D <container name>
# set the passwd and logout
password 
logout 
```

Now we can boot the container in non-interactive mode, either
from the command line or using nspawn files. In either case 
double check that the your bind mounts have the correct permissions 
from inside the container.

```bash 
# for instance attached to a bridge interface br0 
systemd-nspawn -b -D <container name> --network-bridge=br0
# or if you've set up a package cache 
systemd-nspawn -b -D <container name> --network-bridge=br0 --bind=/var/cache/apt/archives
```

Alternately, if you use an nspawn file, then you can use a command 
similar to the following to start it, you'll first need to 
boot the container from the command line and install dbus,
because `machinectl shell` and `machinectl login` won't work 
without dbus. In this case use the following sequence of commands.

```bash
# start the container and login as root
systemd-nspawn -b -D <container name> --network-bridge=br0 
# bring up networking so you can install dbus
# this is also a good time to install and configure locale
apt install dbus locales 
# to configure locale 
dpkg-reconfigure locales 
poweroff
```

After this you can start the container with systemd, when 
using an nspawn file.

```bash 
systemctl start systemd-nspawn@<container name>
```

```text 
# /etc/systemd/nspawn/<container name>.spawn 
[Files] 
# Bind=/var/cache/apt/archives 

[Network] 
bridge=br0 
```

You can use tasksel to install a web-server.

```bash 
# apache2 will immediately be listening on port 80
tasksel install web-server
# enable mod ssl
a2enmod ssl ; systemctl restart apache2
# enable the default ssl test page 
a2ensite /etc/apache2/sites-available/default-ssl.conf
```

You'll be up and running with the default self-signed certs.
