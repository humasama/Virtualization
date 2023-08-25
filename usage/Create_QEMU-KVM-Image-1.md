# Create KVM Image
```
$qemu-img create -f qcow2  ubuntu.qcow2 40G
$qemu-img check ubuntu.qcow2
$qemu-system-x86_64 \
  -m 1024 \
  -hda ./ubuntu.qcow2  \
  -cdrom ./ubuntu-20.04.3-desktop-amd64.iso \
  -enable-kvm
```
Then login the VM:
```
$vncviewer localhost:5900
```
After install ssh-server, we can login via ssh.

# QEMU Command to start VM
1. console, and log in via ```ssh localhost -p 6663```
```
qemu-system-x86_64 -cpu host \
                 -smp cores=2,sockets=1 -enable-kvm -m 2048 \
                 ./100-ubuntu.qcow2 \
                 -object memory-backend-file,id=mem,size=2048M,mem-path=/dev/hugepages,share=on \
                 -numa node,memdev=mem -mem-prealloc \
                 -netdev user,id=n1,hostfwd=tcp::6663-:22 -device e1000,netdev=n1 \
                 -serial mon:stdio -nographic
```
2. graph, and log in via ```vncviewer locahost:5900```
```
$qemu-system-x86_64 -cpu host \
        -smp cores=2,sockets=1 -enable-kvm -m 2048 \
        ./100-ubuntu.qcow2 \
        -object memory-backend-file,id=mem,size=2048M,mem-path=/dev/hugepages,share=on \
        -numa node,memdev=mem -mem-prealloc \
        -netdev user,id=n1,hostfwd=tcp::6664-:22 -device e1000,netdev=n1
```
