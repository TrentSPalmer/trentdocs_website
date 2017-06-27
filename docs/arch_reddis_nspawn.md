# Quick Dirty Reddis Nspawn Container on Arch Linux

## Create a FileSystem

```bash
cd /var/lib/machines
# create a directory
mkdir <container>
# use pacstrap to create a file system
pacstrap -i -c -d <container> base --ignore linux
```
