# Qemu Implementation
This is the third post and let's talk about vhost protocol implementations in Qemu.

## Vhost Protocol Trigger Point
**Qemu** sends vhost protocol messages, when VirtIO device status is set to **```DRIVER_OK```**.
This can happen when VM starts or DPDK starts in guest.
1. VHOST_SET_MEM_TABLE (hdev->vhost_ops->vhost_set_mem_table)
2. VHOST_SET_VRING_ADDR (dev->vhost_ops->vhost_set_vring_addr)
3. VHOST_SET_VRING_KICK (dev->vhost_ops->vhost_set_vring_kick)
4. VHOST_SET_VRING_CALL (dev->vhost_ops->vhost_set_vring_call)

VirtIO spec defines the status below and VirtIO driver uses them to init the device.
```
ACKNOWLEDGE (1)
- Indicates that the guest OS has found the device and recognized it as a valid virtio device. 
DRIVER (2)
- Indicates that the guest OS knows how to drive the device. Note: There could be a significant (or infinite) delay before setting this bit. For example, under Linux, drivers can be loadable modules. 
FAILED (128)
- Indicates that something went wrong in the guest, and it has given up on the device. This could be an internal error, or the driver didn’t like the device for some reason, or even a fatal error during device operation. 
FEATURES_OK (8)
- Indicates that the driver has acknowledged all the features it understands, and feature negotiation is complete. 
DRIVER_OK (4)
- Indicates that the driver is set up and ready to drive the device. 
DEVICE_NEEDS_RESET (64)
- Indicates that the device has experienced an error from which it can’t recover.
```
### 1. VHOST_SET_MEM_TABLE
Qemu sends **```the whole Qemu memory```** information to the backend.

#### vhost-net
When receives the Qemu memory information,
vhost-net fills address translation table **```w/o mmap```**,
as KVM already mmaps Qemu memory when it starts.

vhost-net function is below.
```
vhost_set_memory(struct vhost_dev *d, struct vhost_memory __user *m)
{
	struct vhost_memory mem, *newmem;
	struct vhost_memory_region *region;
	struct vhost_iotlb *newumem, *oldumem;
	unsigned long size = offsetof(struct vhost_memory, regions);

  /* 1. Copy vhost_memory structure */
  copy_from_user(&mem, m, size);
  newmem = kvzalloc(struct_size(newmem, regions, mem.nregions), GFP_KERNEL);
  memcpy(newmem, &mem, size);
  
  /* 2. Copy memory region structures from user-space to kernel space */
  copy_from_user(newmem->regions, m->regions, flex_array_size(newmem, regions, mem.nregions));
  
  /* 3. Add address translation table */
  newumem = iotlb_alloc();
  for (region = newmem->regions; region < newmem->regions + mem.nregions; region++) {
    vhost_iotlb_add_range(newumem,
      region->guest_phys_addr,
      region->guest_phys_addr +
      region->memory_size - 1,
      region->userspace_addr,
      VHOST_MAP_RW);
  }
}
```
#### vhost-user
vhost-user **```mmaps the Qemu memory```** into own address space.

### 2. VHOST_SET_VRING_ADDR
Qemu translated addresses of desc table, avail and used rings set by **```VirtIO driver```** to
**```GPAs```** or **```GVAs```**, and it sends to them to vhost-net.

The important function is **```vhost_dev_has_iommu```**, and it decides GVA or GPA is sent.
This function checks vhost feature **```VIRTIO_F_IOMMU_PLATFORM```** and
**```VirtIO device iommu capability```** which is controlled by users when start Qemu.

Qemu IOMMU default setting:
- VM IOMMU is on.

VirtIO device default setting:
- IOMMU is on in VMX-VM (?);
- IOMMU is on in TD-VM with vhost-net backend;
- IOMMU is off in TD-VM with vhost-user backend.

Backend default setting:
- vhost-user: VIRTIO_F_IOMMU_PLATFORM is not set by default;
- vhost-net: VIRTIO_F_IOMMU_PLATFORM is set I guess (?).

Qemu code is below.
```
static int vhost_dev_has_iommu(struct vhost_dev *dev)
{
    VirtIODevice *vdev = dev->vdev;
    /*
     * For vhost, VIRTIO_F_IOMMU_PLATFORM means the backend support
     * incremental memory mapping API via IOTLB API. For platform that
     * does not have IOMMU, there's no need to enable this feature
     * which may cause unnecessary IOTLB miss/update transactions.
     */
    return virtio_bus_device_iommu_enabled(vdev) &&
           virtio_host_has_feature(vdev, VIRTIO_F_IOMMU_PLATFORM);
}
static void *vhost_memory_map(struct vhost_dev *dev, hwaddr addr,
                              hwaddr *plen, bool is_write)
{
    if (!vhost_dev_has_iommu(dev)) {
        return cpu_physical_memory_map(addr, plen, is_write);
    } else {
        return (void *)(uintptr_t)addr;
    }
}

vhost_virtqueue_start() {
	a = virtio_queue_get_desc_addr(vdev, idx); // vdev->vq[n].vring.desc, it is set by VirtIO Driver.
	vq->avail = vhost_memory_map(dev, a, &l, false);
	vq->used = vhost_memory_map(dev, a, &l, true);
	vq->desc = vhost_memory_map(dev, a, &l, false);

	addr.desc_user_addr = (uint64_t)(unsigned long)vq->desc;
	addr.avail_user_addr = (uint64_t)(unsigned long)vq->avail;
	addr.used_user_addr = (uint64_t)(unsigned long)vq->used;
	vhost_kernel_call(dev, VHOST_SET_VRING_ADDR, addr) {
	  ioctl(dev->opaque, VHOST_SET_VRING_ADDR, addr);
	}
}
```

vhost-net handling function is below.
```
vhost_vring_set_num_addr(argp) {
  copy_from_user(&a, argp, sizeof a);
  vq->log_used = !!(a.flags & (0x1 << VHOST_VRING_F_LOG));
	vq->desc = (void __user *)(unsigned long)a.desc_user_addr;
	vq->avail = (void __user *)(unsigned long)a.avail_user_addr;
	vq->log_addr = a.log_guest_addr;
	vq->used = (void __user *)(unsigned long)a.used_user_addr;
}
```
### 3. VHOST_SET_VRING_BASE/KICK/CALL
VHOST_SET_VRING_BASE: share the init value for index of used and avail rings.

Reference code for Vhost-net vring handling function:
```
vhost_vring_ioctl() {
  if (ioctl == VHOST_SET_VRING_ADDR || ioctl == VHOST_SET_VRING_NUM)
    vhost_vring_set_num_addr();
  switch (ioctl) {
    case VHOST_SET_VRING_BASE:
    case VHOST_GET_VRING_BASE:
    case VHOST_SET_VRING_KICK:
    case VHOST_SET_VRING_CALL:
    ...
}
```
## Address Translation
Qemu supports vIOMMU. With and without vIOMMU, the address translation mechanisms in vhost-user are different.

Some important points need to know:
- The guest’s physical memory space is the memory the guest perceives as physical but, obviously, it is inside QEMU’s process (host) virtual address. When the virtqueue memory region is allocated, it ends up somewhere in the guest’s physical memory space.
- QEMU’s memory management system is aware of where the guest physical memory space resides within its own memory space. Therefore, it is able to translate guest physical addresses to host (QEMU’s) virtual addresses.

### w/ vIOMMU
To enable vIOMMU, users need to do two things:
- set **```VIRTIO_F_IOMMU_PLATFORM```** in vhost-user
- enable vIOMMU in Qemu commands for the VirtIO device (```-device virtio-net-pci,bus=pcie.1,netdev=net0,disable-legacy=on,disable-modern=off,iommu_platform=on,ats=on```)

If vhost-user wants to access the shared memory located by guest IOVA,
it needs to do the following translations:
1. Guest IOVA -> GPA (done by QEMU’s vIOMMU)
2. GPA -> HVA of QEMU process (done by QEMU memory management)
3. **```HVA of QEMU process```** -> **```HVA of vhost-user process```** (done by vhost-user)

To translate Guest IOVA to HVA of QEMU process (#1 and #2),
Vhost-user library (as well as vhost-kernel driver) uses **```IOTLB API```** to request a page translation to QEMU
using a secondary communication channel (**```another unix socket```**) that it creates when IOMMU support is configured.
The IOTLB API receives the request and looks up the address,
first translating the IOVA to the GPA and finally the GPA to the HVA.
It then sends the translated address through the master unix socket back to the vhost-user library.

Finally, the vhost-user library has to make one final translation.
Since it has qemu’s memory mapped into its own,
it has to translate qemu’s HVA to its own HVA and access the shared memory.

I thought vhost-user translates GPA to HVA in data-path, but this is an misunderstanding.
GPA is just a fake concept, and there is no real VM physical memory.
GPA is inside QEMU’s process virtual address, and **```GPA is QEMU process'S HVA```**, in fact.
In the SET_MEM_TABLE message,
QEMU sends its HVA where VM memory resides to vhost-user.

![image](https://github.com/humasama/technologies/assets/6119088/3f3c75a6-1c07-4ec8-8019-901f944ba4b4)

### w/o vIOMMU
When W/O vIOMMU,
VirtIO driver sets GPA (it is QEMU's HVA as explained above) in the vring and vhost-user translates it to HVA.

## Reference
- QEMU/VM memory: https://www.linux-kvm.org/page/Memory
- vhost-user: https://www.redhat.com/en/blog/journey-vhost-users-realm
