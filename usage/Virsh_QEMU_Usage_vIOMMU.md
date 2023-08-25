# vIOMMU and non-IOMMU
There are two terms we need to distinguish: vIOMMU and no-IOMMU.
vIOMMU is a virtualization feature in vCPU,
and a VM can enable or disable vIOMMU via qemu commands.
no-IOMMU is a VFIO mode that can work with both no vIOMMU and vIOMMU VMs.

## vIOMMU VM
this is a virtualization feature that is enabled by qemu by default.
Enabling vIOMMU feature means vCPU supports IOMMU feature,
like host CPU feature IOMMU (VT-d).
Programs in the guest can use VA instead of PA to use devices, like DPDK.
That is, we can bind vNIC to vfio-pci and start DPDK grogram with iova-mode=va.

In the guest supporting vIOMMU,
grograms like DPDK can run in three different modes (https://wiki.qemu.org/Features/VT-d#Use_Case_2:_Guest_Device_Assignment_with_vIOMMU_-_DPDK_Scenario).
- VFIO (VA)
- VFIO no-IOMMU (PA)
- UIO (PA)

### vIOMMU VM QEMU Command
We need two steps to enable a virtio-net device to use vfio with VA (i.e., vfio IOMMU mode):
1. Enable vIOMMU in the VM.
I think QEMU enables vIOMMU by default, so we can leave the below command out(?)
```
-device intel-iommu,intremap=on,device-iotlb=on
```
2. Enable the virtio-net device to use vfio with VA.
```
-device ioh3420,id=pcie.1,chassis=1 \
-device virtio-net-pci,bus=pcie.1,netdev=net0,disable-legacy=on,disable-modern=off,iommu_platform=on,ats=on 
```
See reference in https://wiki.qemu.org/Features/VT-d#Use_Case_2:_Guest_Device_Assignment_with_vIOMMU_-_DPDK_Scenario

### vIOMMU VM xml
```
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <name>test-guest-qcow</name>
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
      <source file='/path/to/qcow2'/>
      <backingStore/>
      <target dev='vda' bus='virtio'/>
    </disk>
    <serial type='pty'>
      <target port='0'/>
    </serial>
    <console type='pty'>
      <target type='serial' port='0'/>
    </console>
    <interface type='vhostuser'>
      <mac address='56:48:4f:53:54:02'/>
      <source type='unix' path='/tmp/sock0' mode='server' />
      <model type='virtio-non-transitional'/>
      <driver name='vhost' iommu='on' ats='on' rx_queue_size='256' />
    </interface>
  </devices>
```
The two commands "**virtio-non-transitional**" and "**iommu='on' ats='on'**"
enables vIOMMU. See xml commands in https://libvirt.org/formatdomain.html#virtio-related-options .

### vHost-user PMD Enablement for vIOMMU
When use vhostpmd, the vhost-user port needs to enable IOMMU by
"**iommu-support=1**".


## No vIOMMU VM
The xml to not enable vIOMMU is below.
```
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <name>test-guest-qcow</name>
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
      <source file='/home/jiayu/vm/libvirt-virsh/run/libvirt-ubuntu-2004.qcow2'/>
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
    <interface type='vhostuser'>
      <mac address='56:48:4f:53:54:02'/>
      <source type='unix' path='/tmp/sock0' mode='server' />
      <model type='virtio'/>
      <driver name='vhost' iommu='off' rx_queue_size='256' />
    </interface>
  </devices>
  <qemu:commandline>
    <qemu:arg value='-netdev'/>
    <qemu:arg value='user,id=mynet.0,net=10.0.10.0/24,hostfwd=tcp::22222-:22,hostfwd=tcp::8000-:8000'/>
    <qemu:arg value='-device'/>
    <qemu:arg value='e1000,netdev=mynet.0'/>
  </qemu:commandline>
</domain>
```
The command "**iommu='off'**" is used to disable vIOMMU.

## no-IOMMU
no-IOMMU is a VFIO mode that can be used in vIOMMU enabled and disabled VMs.

To use the VFIO no-IOMMU mode, we can use the two xml files above.
But in the guest, we need to ```echo 1 > /sys/module/vfio/parameters/enable_unsafe_noiommu_mode```
and bind the vNIC to vfio-pci.

Another command below needs to distinguish (wip).
```
cat /boot/config-5.15.0-generic | grep CONFIG_VFIO_NOIOMMU
```
