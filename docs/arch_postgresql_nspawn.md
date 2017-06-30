# Quick Dirty Postgresql Nspawn Container on Arch Linux

Refer to the [Nspawn](nspawn.md) page for setting up the nspawn container.  
And then refer the [ArchWiki instructions](https://wiki.archlinux.org/index.php/PostgreSQL)
for postgresql.  

You'll want to install postgresql, set a password for the default user `postgres`,
and then login as postgres and initilize the database.  
```bash
pacman -S postgresql
# passwd for postgresql user 
passwd postgres 
# login as postgres 
su -l postgres
# initialize the databse cluster
[postgres]$ initdb --locale $LANG -E UTF8 -D '/var/lib/postgres/data'
```

You'll need to configure `/var/lib/postgres/data/pg_hba.conf` and
`/var/lib/postgres/data/postgresql.conf` for remote access,
presumably with an identd daemon in mind.  The ident daemon will
listen on port 113, not on the machine with the database server,
but it listens from the machine where is the client that remotely
wants to access the database.
