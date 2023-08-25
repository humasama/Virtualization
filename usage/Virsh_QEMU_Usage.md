
# Libvirt HowTo Guide

## XML Sample
```
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <name>test-guest</name>
  <uuid>e0cccc71-1a93-d008-15c0-60155feecb01</uuid>
  <memory unit='MiB'>8192</memory>
  <currentMemory unit='MiB'>8192</currentMemory>
  <vcpu>3</vcpu>
  <os>
    <type arch='x86_64' machine='pc-i440fx-2.9'>hvm</type>
    <boot dev='hd'/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <pae/>
  </features>
  <clock offset='utc'/>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>restart</on_crash>
  <cpu mode='host-passthrough'>
    <topology sockets='1' cores='3' threads='1'/>
    <numa>
      <cell id='0' cpus='0-2' memory='8192' unit='MB' memAccess='shared'/>
    </numa>
  </cpu>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none'/>
      <source file='/path/to/ubuntu.qcow2'/>
      <backingStore/>
      <target dev='vda' bus='virtio'/>
    </disk>
    <serial type='pty'>
      <target port='0'/>
    </serial>
    <console type='pty'>
      <target type='serial' port='0'/>
    </console>
    <graphics type='vnc' port='5900' sharePolicy='allow-exclusive'>
    </graphics>
  </devices>
  <qemu:commandline>
    <qemu:arg value='-netdev'/>
    <qemu:arg value='user,id=mynet.0,net=10.0.10.0/24,hostfwd=tcp::22222-:22,hostfwd=tcp::8000-:8000'/>
    <qemu:arg value='-device'/>
    <qemu:arg value='e1000,netdev=mynet.0'/>
  </qemu:commandline>
</domain>
```
### 1. cpu mode
"cpu mode='host-passthrough'" enables the guest CPU is the same as the host CPU,
including features (e.g. avx512), virtualization (VT-x and VT-d).
"memAccess='share" is needed to enable vhost to access guest memory.

### 2. GUI
```<graphics>``` field specifies vncviewer is used to log in the VM in GUI.
The VNC command is ```vncviewer 0.0.0.0 -p 5900```.

### 3. Network
In the XML file, I use QEMU command to create a e1000 device in the VM.
The e1000 device has IP address, so it can connect to Internet.
In addition, there is a port forwaring, so I ssh the VM via ```ssh xx@localhost -p 22222```.

### Note
To use qemu command in libvirt,
we must use
```<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>```,
as libvirt doesn't support qemu commands by default (i.e., ```<domain type='kvm'>```).

## Virsh Commands
1. Start
```
virsh define ubuntu.xml
virsh start test-guest
 ssh xx@localhost -p 22222
```
2. Shutdown: ```virsh shutdown test-guest```.
3. Destroy: ```virsh destroy test-guest```.
4. Undefine: ```virsh undefine test-guest```.

# References
- network: https://wiki.qemu.org/Documentation/Networking
- xml: https://libvirt.org/formatdomain.html#vhost-user-interface
- xml: https://unix.stackexchange.com/questions/297485/modified-qemu-xml-file-does-not-appear-to-be-used
- vhostuser: https://www.redhat.com/en/blog/hands-vhost-user-warm-welcome-dpdk
