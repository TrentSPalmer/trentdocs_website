# More Notes and Tips for Using LXD

### LXD Server and _Clients_

#####Your LXD hosts can establish a secure client-server relationship very quickly and easily.

```bash
# First enable networking on Both the server and client:
lxc config set core.https_address [::]:8443

# Then on the server, set a password:
lxc config set core.trust_password <something-secure>

# Then on the client, add the server as a remote,
# and enter the password you just created for it:
lxc remote add <server> <ip address>
```


#####Now from the perspective of the client machine, the server is just another remote, same as:  

  * **local** (the default)
  * **ubuntu** (where ubuntu images come from), and 
  * **images** (where lxc images of other distros come from).
  * **clyde** (your host named clyde)


```bash
# command to list remotes
# returns local, images, ubuntu, etc.
lxc remote list

# command to list containers on a remote
lxc list <remote>:
# i.e. for a remote named "black"
lxc list black:

# command to list images on a remote
# i.e. for a remote name "images"
lxc image list images:
# or for a remote named "ubuntu"
lxc image list ubuntu:
# or for a specific image
lxc image list ubuntu:16.04 # or
lxc image list ubuntu:fdceb4d263b9
```

#####Now you can move containers around between servers and clients.

```bash
# launch an ubuntu container from the ubuntu remote
lxc launch ubuntu:16.04 <optional name>
# or from a remote named "black"
lxc launch black:069b95ed3a60 <optional name>
# to list the images that black has available
lxc image list black:

# copy a container from a server named "black"
# to your local client
lxc copy black:jerry <optional name to copy to>
# or from "local" back to "black"
lxc copy jerry black:<optional name to copy to>
# or move
lxc move black:jerry <optional name to copy to>

# or change the default remote from "local" to "black"
lxc remote set-default black
# and then reverse the syntax
# copy a container from a server named "black"
# to your local client
lxc copy jerry local:<optional name to copy to>
# or from "local" back to "black"
lxc copy local:jerry <optional name to copy to>
```

#####Or remote control another LXD server

```bash
# bash shell on container named "jim" running on
# a remote server named "black"
lxc exec black:jim bash

# copy that
lxc copy black:jim black:francine

# snapshot
lxc snapshot black:jim

# delete a snapshot from a remote container
# first get the containers info to see what
# snapshots it has
lxc info black:jim
# and then delete
lxc delete black:jim/snap0

# or rollback/restore,
# slightly different syntax vs "delete"
lxc restore black:jim snap0
```
