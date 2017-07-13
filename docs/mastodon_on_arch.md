# Some Observations About Installing Mastodon on Arch.

## Nginx 

From the [Production Guide](https://github.com/tootsuite/documentation/blob/master/Running-Mastodon/Production-guide.md)
you can copy the example nginx.conf file to `/etc/nginx/sites-enabled/some_arbitrary.conf`,
and then add the following to `/etc/nginx/nginx.conf` in the http section,
this with a fresh install of nginx with the default configuration file.

```bash 
# /etc/nginx/nginx.conf 
http {
    include sites-enabled/*;
}
```

## Installing the Dependancies

```bash
pacman -S certbot nginx libxml2 imagemagick ffmpeg git yarn npm python2 oidentd
```

```bash
# I'm guessing here
pacman -S libpqxx libxslt protobuf protobuf-c
```

* I'm assuming base-devel is installed
* python2 seems to be required to run `yarn install` command later on
* oidentd seems to be a usable replacement for pident
* libpqxx pulls in postgresql-libs
* file is already installed
* curl is already installed
* ruby-build and rbenv are installable from aur
* also postgresql and redis unless, those are in another container or whatever.

## Other Observations

I discovered that between `gem install bundler` and  
`bundle install --deployment --without development test`,
you have to update your environment, with 
`eval "$(rbenv init -)"`, i.e.

```bash 
echo 'eval "$(rbenv init -)"' >> .bashrc
# and then
. ~/.bashrc
```

You have to update your environment more than once, during the
installation.

Presumably you don't ever want to delete the `~/live/Public/` directory
if that is where assets are being stored, but it seems ok to delete 
`~/live/node_modules` and then rerun the `yarn install` command.

In `~/live/.env.production`, `SINGLE_USER_MODE=false` has to be set
to `false` until at least one user is created, or the web service won't 
even start.  (Also `chmod 755 ~/`)
