# Apt Pinning Artful Aardvark Packages in Xenial Xerus

You want to set up apt-pinning so that you can explicitly install packages from
*artful*, on your *xenial* machine, but you also want to be able to issue the command
`apt-get dist-upgrade` and have nothing automatically upgrade from *xenial* to *artful*.

In order to get this to work you have to edit three files. The first file is
`/etc/apt/sources.list`. Make a double length version of the file, with the second
half of the file describing the *artful* equivalent of the *xenial* repos.
Like this.

```conf
# /etc/apt/sources.list
deb http://archive.ubuntu.com/ubuntu xenial main restricted
deb-src http://archive.ubuntu.com/ubuntu xenial main restricted

deb http://archive.ubuntu.com/ubuntu xenial-updates main restricted
deb-src http://archive.ubuntu.com/ubuntu xenial-updates main restricted

deb http://archive.ubuntu.com/ubuntu xenial universe
deb-src http://archive.ubuntu.com/ubuntu xenial universe
deb http://archive.ubuntu.com/ubuntu xenial-updates universe
deb-src http://archive.ubuntu.com/ubuntu xenial-updates universe

deb http://archive.ubuntu.com/ubuntu xenial multiverse
deb-src http://archive.ubuntu.com/ubuntu xenial multiverse
deb http://archive.ubuntu.com/ubuntu xenial-updates multiverse
deb-src http://archive.ubuntu.com/ubuntu xenial-updates multiverse

deb http://archive.ubuntu.com/ubuntu xenial-backports main restricted universe multiverse
deb-src http://archive.ubuntu.com/ubuntu xenial-backports main restricted universe multiverse

deb http://security.ubuntu.com/ubuntu xenial-security main restricted
deb-src http://security.ubuntu.com/ubuntu xenial-security main restricted
deb http://security.ubuntu.com/ubuntu xenial-security universe
deb-src http://security.ubuntu.com/ubuntu xenial-security universe
deb http://security.ubuntu.com/ubuntu xenial-security multiverse
deb-src http://security.ubuntu.com/ubuntu xenial-security multiverse

## Uncomment the following two lines to add software from Canonical's
## 'partner' repository.
## This software is not part of Ubuntu, but is offered by Canonical and the
## respective vendors as a service to Ubuntu users.
# deb http://archive.canonical.com/ubuntu xenial partner
# deb-src http://archive.canonical.com/ubuntu xenial partner

deb http://archive.ubuntu.com/ubuntu artful main restricted
deb-src http://archive.ubuntu.com/ubuntu artful main restricted

deb http://archive.ubuntu.com/ubuntu artful-updates main restricted
deb-src http://archive.ubuntu.com/ubuntu artful-updates main restricted

deb http://archive.ubuntu.com/ubuntu artful universe
deb-src http://archive.ubuntu.com/ubuntu artful universe
deb http://archive.ubuntu.com/ubuntu artful-updates universe
deb-src http://archive.ubuntu.com/ubuntu artful-updates universe

deb http://archive.ubuntu.com/ubuntu artful multiverse
deb-src http://archive.ubuntu.com/ubuntu artful multiverse
deb http://archive.ubuntu.com/ubuntu artful-updates multiverse
deb-src http://archive.ubuntu.com/ubuntu artful-updates multiverse

deb http://archive.ubuntu.com/ubuntu artful-backports main restricted universe multiverse
deb-src http://archive.ubuntu.com/ubuntu artful-backports main restricted universe multiverse

deb http://security.ubuntu.com/ubuntu artful-security main restricted
deb-src http://security.ubuntu.com/ubuntu artful-security main restricted
deb http://security.ubuntu.com/ubuntu artful-security universe
deb-src http://security.ubuntu.com/ubuntu artful-security universe
deb http://security.ubuntu.com/ubuntu artful-security multiverse
```

Now create a new file `/etc/apt/preferences.d/xenial` with the
following content.

```conf
Package: *
Pin: release a=xenial
Pin-Priority: 900
```

And create one more file `/etc/apt/preferences.d/artful` with the
following content.

```conf
Package: *
Pin: release a=artful
Pin-Priority: 300
```

Actually, I'm not entirely certain these are the optimal apt-pinning
priority numbers. There's a little bit of art to apt-pinning.

So you can verify that nothing will automatically upgrade with the
following command.

```bash
# the result of this command should be that nothing upgrades
apt-get dist-upgrade
```

But let's suppose that you want to explicitly install a package, and
hopefully the upgraded dependancies which it needs from *artful*.
`apt-cache madison` is a useful command.

```bash
apt-cache madison weather-util
# outputs the following
weather-util |      2.3-2 | http://archive.ubuntu.com/ubuntu artful/universe amd64 Packages
weather-util |      2.0-1 | http://archive.ubuntu.com/ubuntu xenial/universe amd64 Packages
weather-util |      2.0-1 | http://archive.ubuntu.com/ubuntu xenial/universe Sources
weather-util |      2.3-2 | http://archive.ubuntu.com/ubuntu artful/universe Sources
```

As you can see, two different version of *weather-util* are available (as
well as two different source versions), one each from the *xenial*,
and the *artful* repos.

But if you type `apt-get install weather-util`, the old version from the *xenial*
repo will be installed. The intended behaviour is entirely a matter of getting
the apt-pinning priority numbers correct.

To explicitly install the newer version of *weather-util*, and perhaps more
importantly it's upgraded *weather-util-data* dependancy, use the following command.

```bash
apt-get -t artful install weather-util
```

But hold on, HOLD ON! The above command doesn't actually confirm what version is
going to be installed, and you'd like to have one last look things over, so add
the `-V` flag to your `apt-get` command.

```bash
root@xhost:~# apt-get -t artful install weather-util -V
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following additional packages will be installed:
  weather-util-data (2.3-2)
The following NEW packages will be installed:
  weather-util (2.3-2)
     weather-util-data (2.3-2)
     0 upgraded, 2 newly installed, 0 to remove and 389 not upgraded.
     Need to get 0 B/3375 kB of archives.
     After this operation, 3557 kB of additional disk space will be used.
     Do you want to continue? [Y/n] 
```

That's what you're looking for.
