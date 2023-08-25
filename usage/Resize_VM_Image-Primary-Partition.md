# HowTO - Increase VM Disk Size

## Image Disk Status Before

1. First, we check image information by qemu-img.

![image](https://user-images.githubusercontent.com/6119088/233771074-a643cd34-184b-4808-9b06-ed2e83a93289.png)

2. Second, start VM and get the aim partition that we want to resize.

- First, we get disk name by ```lsblk```. As shown below, the disk name is /dev/vda.

![image](https://user-images.githubusercontent.com/6119088/233771278-823839b3-c603-440e-8ec3-302821b1521f.png)

- Second, get the partition that Linux is installed by the command below.
```
fdisk /dev/vda

//In the command window of fdisk
p
```
From the output of ```p```,
we can see "/dev/vda1" is where Linux locates and it is a **primary partition**.

## Goals
Our goal are:
1. increase the whole disk size from 22GB to 42GB;
2. Add 20GB for the partition /dev/vda1, which is a primary partition.

Before resize, let's check the size of /dev/vda1 currently by ```df -H```.
As we can see, it is 23GB.

![image](https://user-images.githubusercontent.com/6119088/233771227-33f94eff-0351-4bea-8d6b-4cc58f9cee55.png)

## Steps
1. In host:
```
// Stop the running VM
virsh destroy ubuntu_qcow2_img

// Increase VM image disk size
qemu-img resize ubuntu_qcow2_img +20G

// The virtual size is increased by 20GB
qemu-img info ubuntu_qcow2_img
```

2. In VM, we delete the old partition /dev/vda1 and create a new /dev/vda1 by the following 4 steps.

- First, we enter the command window by running ```fdisk /dev/vda```.
- Second, we delete the old /dev/vda1 by inputting ```d``` -> ```1```.

![image](https://user-images.githubusercontent.com/6119088/233771779-f4ee238e-b140-471f-bf8a-f003797d8b8b.png)

- Third, we create a new /dev/vda1 by inputting ```n``` -> ```1```. In this step, we use the default First sector and Last sector value.

![image](https://user-images.githubusercontent.com/6119088/233771824-ee526d1f-c2c5-47b6-a110-685bca109bf4.png)

- Finally, we save the change by inputting ```w```.

![image](https://user-images.githubusercontent.com/6119088/233771887-f0661d3b-e88e-4a49-a588-b2e4976efde5.png)

4. Restrat the VM, apply the new disk space for /dev/vda1 by running ```resize2fs /dev/vda1```.

## Steps for Extended Partition (WIP)

```
resize2fs /dev/vda5
resize2fs /dev/vda2
```

## Reference
- https://woshub.com/kvm-expand-shrink-vm-disk/
- https://www.baeldung.com/linux/extend-logical-extended-partitions-fdisk
