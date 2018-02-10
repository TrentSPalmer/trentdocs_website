# Stupid Package Manager Tricks

## apt, apt-get ,aptitude, dpkg

Wait what was that list of suggested packages?

`apt-cache depends <package>`
or
`apt-cache depends <package> | grep -i Suggests`

What versions of a package are available, (based on currently configured repositories)?

`apt-cache madison <package>`
