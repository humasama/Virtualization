# QEMU Commands

## Install QEMU from source
```
git clone https://gitlab.com/qemu-project/qemu.git
./configure --target-list=x86_64-softmmu --prefix=/usr/local/qemu-6.1.0
make -j20
// QEMU will be installed under /usr/local/qqemu-6.1.0
make install
// make clean
```

## vhostuser+vIOMMU+VirtioPMD
1. QEMU Command
- Enable platform IOMMU for VM (vIOMMU): **```-M q35,accel=kvm,kernel-irqchip=split```**, **```-device intel-iommu,intremap=on,device-iotlb=on```**
- Enable VirtIO device to use VIOMMU: **```disable-legacy=on,disable-modern=off,iommu_platform=on,ats=on```**
```
/usr/local/qemu-6.1.0/bin/qemu-system-x86_64 -cpu host \
-smp cores=2,sockets=1 \
-enable-kvm -m 8192 \
-M q35,accel=kvm,kernel-irqchip=split \
-drive file=./100-ubuntu.qcow2 \
-device intel-iommu,intremap=on,device-iotlb=on \
-device ioh3420,id=pcie.1,chassis=1 \
-object memory-backend-file,id=mem,size=8192M,mem-path=/dev/hugepages,share=on \
-numa node,memdev=mem -mem-prealloc \
-netdev user,id=n1,hostfwd=tcp::6664-:22 -device e1000,netdev=n1 \
-chardev socket,id=chr0,path=/tmp/sock0,server=on \
-netdev type=vhost-user,id=net0,chardev=chr0,vhostforce=on \
-device virtio-net-pci,bus=pcie.1,netdev=net0,disable-legacy=on,disable-modern=off,iommu_platform=on,ats=on \
-serial mon:stdio -nographic
```
2. vhostuser Command: ```--vdev "...,iommu-support=1"```
3. VM settings:
- Update grub file below and run ```grub-update```.
- Don't enable no-iommu. Directly bind VirtIO device to vfio-pci, and you will see guest testpmd uses VA.
```
//etc/default/grub
GRUB_CMDLINE_LINUX="iommu=pt intel_iommu=on"
```

## MISC
```
qemu-system-x86_64 -cpu host \
-smp cores=2,sockets=1 -enable-kvm -m 2048 \
./ubuntu.qcow2 \
-object memory-backend-file,id=mem,size=2048M,mem-path=/dev/hugepages,share=on \
-numa node,memdev=mem -mem-prealloc \
-netdev user,id=n1,hostfwd=tcp::6664-:22 -device e1000,netdev=n1 \
-vnc :5
```

The VNC command to login VM is ```vncviewer 0.0.0.0:5```.
