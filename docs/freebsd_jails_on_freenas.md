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

## running gitit under the supervision of supervisord
py27-supervisor and hs-gitit are available as pkg install, if you want to
run a gitit wiki.

gitit doesn't come with an init service. To generate a sample config,
run `gitit --print-default-config > gitit.conf`, and then if you want
you can reference gitit.conf by passing gitit the *-f* flag.

So for instance, after you install supervisord, add something like the
following to the end of `/usr/local/etc/supervisord.conf`, and create
the directory `/var/log/supervisor/`.

```conf
[program:gitit]
user=<user>
directory=/path/to/wikidata/directory/
command=/usr/local/bin/gitit -f /usr/local/etc/gitit.conf
stdout_logfile=/var/log/supervisor/%(program_name)s.log
stderr_logfile=/var/log/supervisor/%(program_name)s.log
autorestart=true
```

supervisord is a service you can enable in
`/etc/rc.conf`

```conf
# /etc/rc.conf
supervisord_enable="YES"
```

and then start with `service supervisord start`
when you get supervisord running, you can start a
supervisorctl shell, i.e.

```sh
supervisorctl
supervisor> status
# outputs
gitit                            RUNNING   pid 98057, uptime 0:32:27
supervisor> start/restart/stop gitit
supervisor> exit
```

But there is one other little detail, in that when you try to
run gitit as a daemon like this, on FreeBSD it will fail because it can't
find git. But the symlink solution is easy enough.

```csh
ln -s /usr/local/bin/git /usr/bin/
```

And you might as well stick a reverse proxy in front of it. Assuming
you configure gitit listen only on localhost:5001, install nginx.
`pkg install nginx`

enable nginx in /etc/rc.conf

```conf
nginx_enable="YES"
```

Then, in the file `/usr/local/etc/nginx/nginx.conf` change the location "*/*"
so that it looks like this.

```nginx
{
.....
        location / {
            # root   /usr/local/www/nginx;
            # index  index.html index.htm;
                proxy_pass http://127.0.0.1:5001;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
....
}
```

and then start nginx `service nginx start`
