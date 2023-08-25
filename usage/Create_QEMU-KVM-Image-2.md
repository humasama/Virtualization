# Create VM Image

In this post, I will virt-install of Libvirt to create a image.


# 1. Install KVM/QEMU and Libvirt
## Commands
```
apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst virt-manager
systemctl is-active libvirtd
usermod -aG libvirt $(whoami)
usermod -aG libvirt-qemu $(whoami)
```

## Check status
```
systemctl status libvirtd.service
systemctl status qemu-kvm.service
brctl show
```

## Debug
1. Find qemu related packages:
```
find / -name qemu*
```
## Uninstall Package
The way to uninstall all qemu related packages is to find all packages first,
then manually remove all.
1. Find and remove one package
```
dpkg --list | grep qemu
apt-get purge "the-package-name"
```
2. Clean environment
```
apt-get autoremove
apt-get clean
```
## References
- https://linuxize.com/post/how-to-install-kvm-on-ubuntu-20-04/
- Uninstall package: https://askubuntu.com/questions/151941/how-can-you-completely-remove-a-package


# 2. Create VM Image
I use virt-install to install OS on a created qcow2 image.
```
virt-install \
--name ctest1 \
--ram 2048 \
--disk path=/path/to/ubuntu.qcow2,size=60 \
--vcpus 2 \
--os-type linux \
--os-variant ubuntu20.04 \
--network network:default \
--graphics vnc,listen=0.0.0.0,port=5900 \
--console pty,target_type=serial \
--location /path/to/ubuntu-20.04.6-desktop-amd64.iso \
--extra-args 'console=ttyS0,115200n8 serial' \
--force --debug
```
virt-install will start the iso first, then we log in via ```vncviewer 0.0.0.0 -p 5900``` to install OS
on the ubuntu.qcow2 disk.

## Errors
When running the above command, I am in root priviledge.
And I meet one error "MoTTY X11 proxy: Unsupported authorisation protocol".
I was pretty familiar with this error when launch qemu, and at that time,
OS in the server I ran was server version (not desktop version) that has no GUI...
Ok I meet the error again, even the server's OS is desktop version...

But luckily, I found one solution (https://superuser.com/questions/610084/putty-x11-proxy-wrong-authorisation-protocol-attempted).
I copied non-root's ~/.Xauthority file to root's ~/.Xauthority.


## References
- https://fabianlee.org/2018/10/06/kvm-creating-an-ubuntu-vm-with-console-only-access/
