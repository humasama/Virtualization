# VirtIO PMD - Device HW Register
This is the first post of a serial that introduces how to implement own VirtIO net device PMD.

## Device Configuration Sturctures
The virtio device configuration layout includes 4 device specific structures
and one standard PCI configuration space.
The five strucures is emulated in Guest memory by Qemu.

- Common configuration
- ISR Status
- Device-specific configuration (useless)
- Notifications
- PCI configuration space (not used in VirtIO PMD)

**Every structure** is specified by **```struct virtio_pci_cap```** which is located in the **capability list** in PCI configuration space of the device.
```
struct virtio_pci_cap { 
        u8 cap_vndr;    /* Generic PCI field: PCI_CAP_ID_VNDR */ 
        u8 cap_next;    /* Generic PCI field: next ptr. */ 
        u8 cap_len;     /* Generic PCI field: capability length */ 
        u8 cfg_type;    /* Identifies the structure. */ 
        u8 bar;         /* the BAR index that the structure resides */
        u8 padding[3];  /* Pad to full dword. */ 
        le32 offset;    /* Offset within bar. */
        le32 length;    /* Length of the structure, in bytes. */ 
};

// Values of cfg_type
/* Common configuration */ 
#define VIRTIO_PCI_CAP_COMMON_CFG        1 
/* Notifications */ 
#define VIRTIO_PCI_CAP_NOTIFY_CFG        2 
/* ISR Status */ 
#define VIRTIO_PCI_CAP_ISR_CFG           3 
/* Device specific configuration */ 
#define VIRTIO_PCI_CAP_DEVICE_CFG        4 
/* PCI configuration access */ 
#define VIRTIO_PCI_CAP_PCI_CFG           5
```

### 1. Obtain virtio_pci_cap
As virtio_pci_cap structure resides in the PCI config space,
we need to first access its PCI config space either via VFIO.
This is done by pread/pwrite the device file represented by VFIO (https://github.com/humasama/technologies/blob/main/Kernel/VFIO_Configure_Device.md).

```
// Get offset of virtio_pci_cap structure within the device file.
uint8_t pos;
pread(device, &pos, sizeof(pos), VFIO_GET_REGION_ADDR(VFIO_PCI_CONFIG_REGION_INDEX) + PCI_CAPABILITY_LIST);
 
// The virtio_pci_cap structur is in a linked-list, so we need to access one-by-one,
// until find the one we need.
{
  struct virtio_pci_cap cap;
  pread(device, &cap, sizeof(cap), VFIO_GET_REGION_ADDR(VFIO_PCI_CONFIG_REGION_INDEX) + pos);
  pos = cap.next;
}
```

### 2. Access Configure Structures via virtio_pci_cap
VirtIO configuration structures are stored in Guest memory by Qemu when emulates a virtio-net device.

There are two methods,
read/write mmaped memory and file read/write,
to access the structure.
For both methods,
we need to find the **```offset```** of the target structure **within the file**
of device (Note that Linux represents every device as a file and it has an unique FD).
The structure offset can be calculated by **```virtio_pci_cap.bar```** and **```virtio_pci_cap.offset```**.

In addition, any read/write to the configuration structure causes **```VM exit```**,
as they writes/reads a MMIO space in VM.
After VM exit, KVM notifies Qemu to do the real VirtIO device configuration.

1. MMIO Method
```
// Mmap address space for device memory specified by BAR.
ioctl(vfio_dev_fd, VFIO_DEVICE_GET_REGION_INFO, region);
dev->BARs[i] = mmap(NULL, region->size, ..., vfio_dev_fd, region->offset);

// Calculate the configure structure address in the given BAR.
addr = dev->BARs[virtio_pci_cap.bar] + virtio_pci_cap.offset
```

2. File read/write
```
// the bar index is equal to the VFIO region index.
#define VFIO_GET_REGION_ADDR(x) ((uint64_t) x << 40ULL)
offset = VFIO_GET_REGION_ADDR(virtio_pci_cap.bar) + cap.offset;
pread(vfio_dev_fd, ..., offset);
```

Sample code see: ```read_pci_config()``` in https://github.com/humasama/vfio/blob/master/vfio-noiommu-pci-device-open.c.

## Useful Configuration Structures
Although there are 5 configuration structures in VirtIO,
only three are used, to my knowledge.
- VIRTIO_PCI_CAP_COMMON_CFG: common device configure
- VIRTIO_PCI_CAP_NOTIFY_CFG: VirtIO notification to vhost
- VIRTIO_PCI_CAP_DEVICE_CFG: VirtIO capabilities


### VIRTIO_PCI_CAP_COMMON_CFG
```
struct virtio_pci_common_cfg { 
        /* About the whole device. */ 
        le32 device_feature_select;     /* read-write */ 
        le32 device_feature;            /* read-only for driver */ 
        le32 driver_feature_select;     /* read-write */ 
        le32 driver_feature;            /* read-write */ 
        le16 msix_config;               /* read-write */ 
        le16 num_queues;                /* read-only for driver */ 
        u8 device_status;               /* read-write */ 
        u8 config_generation;           /* read-only for driver */ 
 
        /* About a specific virtqueue. */ 
        le16 queue_select;              /* read-write */ 
        le16 queue_size;                /* read-write, power of 2, or 0. */ 
        le16 queue_msix_vector;         /* read-write */ 
        le16 queue_enable;              /* read-write */ 
        le16 queue_notify_off;          /* read-only for driver */ 
        le64 queue_desc;                /* read-write */ 
        le64 queue_avail;               /* read-write */ 
        le64 queue_used;                /* read-write */ 
};
```
### VIRTIO_PCI_CAP_NOTIFY_CFG
```
struct virtio_pci_notify_cap { 
        struct virtio_pci_cap cap; 
        le32 notify_off_multiplier; /* Multiplier for queue_notify_off. */ 
};
```
VIRTIO_PCI_CAP_NOTIFY_CFG and **```virtio_pci_common_cfg.queue_notify_off```** are
used together to calculate the VIRTIO_PCI_QUEUE_NOTIFY register address for every vring.
The driver accesses VIRTIO_PCI_QUEUE_NOTIFY register to notify the backend in data-path.
So any writes to it will cause VM exit.
### VIRTIO_PCI_CAP_DEVICE_CFG
Device capabilities, like mac address, reported by Qemu.
```
struct virtio_net_config {
        /* The config defining mac address (if VIRTIO_NET_F_MAC) */
        uint8_t    mac[RTE_ETHER_ADDR_LEN];
        /* See VIRTIO_NET_F_STATUS and VIRTIO_NET_S_* above */
        uint16_t   status;
        uint16_t   max_virtqueue_pairs;
        uint16_t   mtu;
        /*
         * speed, in units of 1Mb. All values 0 to INT_MAX are legal.
         * Any other value stands for unknown.
         */
        uint32_t speed;
        /*
         * 0x00 - half duplex
         * 0x01 - full duplex
         * Any other value stands for unknown.
         */
        uint8_t duplex;
        uint8_t rss_max_key_size;
        uint16_t rss_max_indirection_table_length;
        uint32_t supported_hash_types;
}
```
### 4. Others
VIRTIO_PCI_CAP_PCI_CFG is not used, and VIRTIO_PCI_CAP_ISR_CFG detail is below.
```
struct virtio_pci_dev {
	...
	uint8_t *isr;
	...
}
```
