# Stupid KVM Tricks

## virt-install ubuntu16.04

Create the disk image

`qemu-img create -f qcow2 /var/lib/libvirt/images/xenial.qcow2 20G`

Command to run the install

```bash
virt-install \
    --name xenial \
    --ram 4096 \
    --disk path=/var/lib/libvirt/images/xenial.qcow2,size=20 \
    --vcpus 4 \
    --os-type linux \
    --os-variant ubuntu16.04 \
    --network bridge=br0 \
    --graphics none \
    --console pty,target_type=serial \
    --location ./ubuntu-16.04.3-server-amd64.iso \
    --extra-args 'console=ttyS0,115200n8 serial'
```
## virt-install Arch Linux
The `--extra-args` option lets you use a serial console. But the
`--extra-args` option only works if you also use an `--location`
option. But the `--location` option can only be used with certain isos.
So use `--cdrom` instead of `--location`, drop the `--extra-args`,
and instruct the kernel to boot with a serial console with a parameter
at the boot splash screen.

```bash
qemu-img create -f qcow2 /var/lib/libvirt/images/arch.qcow2 20G

virt-install --name arch --ram 4096 \
  --disk path=/var/lib/libvirt/images/arch.qcow2,size=20 \
  --vcpus 2 \
  --os-type linux \
  --os-variant ubuntu16.04 \ 
  --network bridge=virbr0 \
  --graphics none \ 
  --console pty,target_type=serial \
  --cdrom /var/lib/libvirt/images/archlinux-2018.02.01-x86_64.iso
```

the arch boot splash screen will appear in your terminal and you can 
tap the "tab" key to edit boot parameters

add "console=ttyS0" to kernel command line parameters

before
```bash
> .linux boot/x86_64/vmlinuz archisobasedir=arch archisolabel=ARCH_201802 initrd=boot/intel_ucode.img,boot/x86_64/archiso.img
```
after
```bash
> .linux boot/x86_64/vmlinuz archisobasedir=arch archisolabel=ARCH_201802 initrd=boot/intel_ucode.img,boot/x86_64/archiso.img console=ttyS0
```
```bash

arch boots ...
...
...
...

root@archiso ~ # lsblk
NAME  MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0   7:0    0  432M  1 loop /run/archiso/sfs/airootfs
sr0    11:0    1  539M  0 rom  /run/archiso/bootmnt
vda   254:0    0   20G  0 disk 
root@archiso ~ # 
```
## Change the Network Interface
br0 gets addresses from the network router, but what if you want
your vm to have be on the virbr0 192.168.122.0/24 subnet?

`virsh edit xenial`

And then 'J' all the way down to the bottom, change the interface name from br0 to
virbr0, 

`virsh start xenial`

and then look for the machine with nmap

`nmap -sn 192.168.122.0/24`

## Clone the VM

In this case we don't have to pre-allocate the disk image because virt-clone will do that
for us.

```bash
virt-clone --original xenial --name xenial-clone \
    --file /var/lib/libvirt/images/xenial-clone.qcow2
```

## Clone the VM to another Machine

First dump the xml that defines the virtual machine.

```bash
virsh dumpxml xenial > xenial.xml
```

Then copy both `xenial.xml` and `xenial.qcow2` to the new host machine. On the new kvm
host you'll want to at least make sure your vm has the correct CPU architecture.
The command to get a list of supported kvm cpu architectures is:

```bash
virsh cpu-models <arch>
# i.e.
virsh cpu-models x86_64
```

After you edit `xenial.xml` and update the correct cpu architecture, mv `xenial.qcow2`
to `/var/lib/libvirt/images/`, clone it. `virt-clone` will handle generating new
mac addresses for the network interfaces.

```xml
  <cpu mode='custom' match='exact'>
    <model fallback='allow'>Haswell-noTSX</model>
  </cpu>
# i.e. change to above to
  <cpu mode='custom' match='exact'>
    <model fallback='allow'>SandyBridge</model>
  </cpu>

```


```bash
virt-clone --original-xml xenial.xml --name xenial-clone \
    --file /var/lib/libvirt/images/xenial-clone.qcow2
```

## What is the os-type and os-variant type names?

`osinfo-query os`

## misc

* Start the vm `virsh start xenial`  
* List all the vms `virsh list --all`  
* Stop the vm `virsh destroy xenial`   
* Delete the vm `virsh undefine xenial`  

## virsh help

The `virsh help` command returns a long chart of help information. But each section has
a keyword.

Take for instance the command `virsh help monitor`. From this we
see the `domiflist` subcommand (among others). Unfortunately `domifaddr` doesn't seem to
work on the Ubuntu:16.04 host, but there are other ways to find the ip address of
a virtual machine.

So now if you want to see what host interface the vm `xenial` is attached to,
type. 

```bash
virsh domiflist xenial
```

which returns:
```bash
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet1      bridge     virbr0     virtio      52:54:00:58:bf:75
```

So now we can find the address of virbr0 on the host machine.

```bash
ifconfig virbr0
```
which returns:

```bash
virbr0    Link encap:Ethernet  HWaddr 52:54:00:38:87:38  
          inet addr:192.168.122.1  Bcast:192.168.122.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:1351 errors:0 dropped:0 overruns:0 frame:0
          TX packets:3037 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:232346 (232.3 KB)  TX bytes:502916 (502.9 KB)
```

and thus we know what subnet to scan with nmap to find the ip address of the vm

```bash
nmap -sn 192.168.122.0/24
```

## Snapshots

Create snapshot of vm `dcing`

```bash
virsh snapshot-create-as --domain dcing --name dcing-snap0
```
But you don't need to name your snapshots because they are listed by time.

```bash
virsh snapshot-create --domain dcing
```
List snapshots for vm `dcing`

```bash
virsh snapshot-list --domain dcing

 Name                 Creation Time             State
------------------------------------------------------------
 1518366561           2018-02-11 08:29:21 -0800 shutoff
 dcing-snap0          2018-02-11 08:22:57 -0800 shutoff
```
Revert dcing to snap0

```bash
virsh snapshot-revert --domain dcing --snapshotname dcing-snap0
```
Delete snapshot

```bash
virsh snapshot-delete --domain dcing --snapshotname dcing-snap0
```

