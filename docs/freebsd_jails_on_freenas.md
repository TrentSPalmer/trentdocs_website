# FreeBSD Jails on FreeNAS
Mostly a personal distillation for getting a FreeBSD
Jail up and running on FreeNAS.

## In The FreeNAS WebGui, Create A New Jail

The default networking configuration, will give
your jail an ip address on the lan. For now, I've
decided to just share a pkg cache with each jail.
Navigate to `Jails -> Storage -> Add Storage` and
add the `pkg` storage directory to `/var/cache/pkg`
inside the jail.  

For instance, on my local FreeNAS server,
the pkg directory is at /mnt/VolumeOne/pkg/.

If you ssh into the host server, you can type the command
`jls`, to list the jails. Based on the output of the
command `jls`, you can get a shell with `jexec <jail number>`
of `jexec <jail hostname>`.

### updating

How about the command `pkg audit -F`? Downloads a
list of known security issues and checks your system
against that.

I would recommend, to myself anyway, to shell into
the new jail with `jexec`, run `pkg upgrade` to install any new packages,
and then from the FreeNAS webgui, restart the jail. Although
the restarted jail will have a new jail number as reported by
the `jls` command.

### locale

When you use `jexec` to get a shell, you get an environment
with an utf_8 locale. Not so if you ssh into the new jail.
For this put the following contents into ~/.login_conf

```conf
# ~/.login_conf
me:\
        :charset=UTF-8:\
        :lang=en_US.UTF-8:\
        :setenv=LC_COLLATE=C:
```

### ssh

To get ssh running, edit `/etc/rc.conf` inside the jail.

```conf
# /etc/rc.conf
sshd_enable="YES"
```

To start sshd immediately, make any necessary edits to
/etc/ssh/sshd_config, and run the following command.

```csh
service sshd start
```

## Byobu

You'll need newt to configure byobu, and if you don't install tmux
then screen will become the backend.

```csh
pkg install byobu tmux newt
```

If you execute `byobu-config`, by pressing *f9*, the
following options seem to work. Some options, of course,
will prevent others from working so you have to enable them
one at a time to see what happens.

* date
* disk
* distro
* hostname
* ip address
* load_average
* logo
* time
* uptime
* users
* whoami

## vim

Via pkg, there are two options: vim and vim-lite. Note vim will pull
in a whole bunch of gui dependancies, but vim-lite is not build with python.

For instance, powerline will not work with vim-lite because it's not built with
python. Also, vim-youcompleteme will not work with vim-lite. However, lightline
will work with vim-lite, and VimCompletesMe will work with vim-lite.

To get lightline working update $TERM

```config
# ~/.config/fish/config.fish
export TERM=xterm-256color
```

And vimrc

```vim
# ~/.vimrc
set ls=2
```

Another option is to build vim from source via ports. You can prevent vim
from pulling in a bunch of gui dependancies with the following in /etc/make.conf.

```conf
# /etc/make.conf
WITHOUT_X11=yes
```

And then when you compile vim from ports, run `make config` where you can enable
python.

## python
For python3 virtualenv

```csh
virtualenv-3.6 <directory>
```