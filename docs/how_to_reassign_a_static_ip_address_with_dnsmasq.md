# How To Reassign a Static ip address with dnsmasq

On your router you can assign static ip addresses for various machines
in your network, by writing the reservations in the file `/etc/dnsmasq.conf`.

These will be in the form as below.

`dhcp-host=<mac address>,<ip address>`

So here's how you transfer an existing static ip address assignment to
a new client machine. Begin by editting the file `/etc/dnsmasq.conf` on
your router, and update the mac address associated with the intended
ip address.

Next, temporarily stop dnsmasq.

```bash
systemctl stop dnsmasq
```

Next shutdown networking on the new client machine. Shutting the machine down might work,
or the command `dhclient -v -r` might get the job done (you will lose the connection).

Now on the router, edit the file `/var/lib/misc/dnsmasq.leases`, and delete the pre-existing
lease for the old client machine that will no longer exist.

Restart dnsmasq on the router,  
and then restart networking on the new client machine.
