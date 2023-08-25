# VirtIO Device with Different Backends
This post introduces how to create VirtIO device with different backends: QEMU, vhost-net, and vhost-user.

## Prepare - Create Network
Before creating a VirtIO device for VM, we need to create a **```network```** to enable the VirtIO device to connect with outside.
When creating the VirtIO device, we need to explicitly add it to the network.

QEMU has a default networking called "default" and its XML file locates in /usr/share/libvirt/networks/default.xml.
```
<network>
  <name>default</name>
  <bridge name='virbr0'/>
  <forward/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>
```

## Create VirtIO with QEMU Backend
The commands below are used in VM XML file to create a VirtIO network device with QEMU backend.
Note that QEMU backend is a **```user-space backend```**.
The QEMU backend is specified by **```driver name='qemu'```**,
and the network is specified by **```source network='default' bridge='virbr0'```**.
```
<interface type='network'>
  <mac address="52:54:00:13:34:44"/>
  <source network='default' bridge='virbr0'/>
  <target dev='vnet1'/>
  <model type='virtio'/>
  <alias name='net1'/>
  <driver name='qemu' iommu = 'on'/>
  <address type='pci' domain='0x0000' bus='0x02' slot='0x00' function='0x0'/>
</interface>
```

We can use the following command to inspect host QEMU.
As you can see the results below,
QEMU opens a FD for /dev/kvm and /dev/net/tun.
The later is to communicate outside for the VirtIO device.
In addition, there is no FD opened by QEMU points to /dev/vhost-net,
which means vhost-net is not used.
```
#ls -lh /proc/$(pgrep qemu)/fd | grep '/dev'
lrwx------ 1 root root 64 May 25 03:50 0 -> /dev/null
lrwx------ 1 root root 64 May 25 03:50 3 -> /dev/ptmx
lrwx------ 1 root root 64 May 25 03:50 34 -> /dev/net/tun
lrwx------ 1 root root 64 May 25 03:50 9 -> /dev/kvm
```

## Create VirtIO with Vhost-net Backend
The vhost-net backend (kernel space backend) is specified by **```driver name='vhost'```**.
```
<interface type='network'>
  <mac address="52:54:00:13:34:44"/>
  <source network='default' bridge='virbr0'/>
  <target dev='vnet1'/>
  <model type='virtio'/>
  <alias name='net1'/>
  <driver name='vhost' iommu = 'on'/>
  <address type='pci' domain='0x0000' bus='0x02' slot='0x00' function='0x0'/>
</interface>
```

When inspect host QEMU, you can see results below.
QEMU opens a FD to point to /dev/vhost-net,
which means the backend is vhost-net.
```
#ls -lh /proc/$(pgrep qemu)/fd | grep '/dev'
lrwx------ 1 root root 64 May 25 03:54 0 -> /dev/null
lrwx------ 1 root root 64 May 25 03:54 3 -> /dev/ptmx
lrwx------ 1 root root 64 May 25 03:54 34 -> /dev/net/tun
lrwx------ 1 root root 64 May 25 03:54 36 -> /dev/vhost-net
lrwx------ 1 root root 64 May 25 03:54 9 -> /dev/kvm
```
## Reference
- vhost-net: https://www.redhat.com/en/blog/hands-vhost-net-do-or-do-not-there-no-try
