# DPDK and Kernel VirtIO Driver Comparison

## Feature Definition

| Kernel | DPDK | Mean | Value | Default Enabled |
|:--------------:|:-------------:|:-----:|:-----:|:-----:|
| VIRTIO_F_ACCESS_PLATFORM | VIRTIO_F_IOMMU_PLATFORM | Use platform DMA tools to access the memory | 33 | Yes|

# Features

## 1. VIRTIO_NET_F_CTRL_VQ
- VirtIO feature: VIRTIO_NET_F_CTRL_VQ (17)
- **```Control channel```** is used to communicate **```ethernet device capabilities```**,
like setting MAC address,
between VirtIO driver and vhost backend.


For example, virtio_mac_addr_set() is called by rte_eth_dev_start()
and control vring is used to send updated MAC address to vhost backend.
```
int
rte_eth_dev_start(uint16_t port_id)
{
  ...
  eth_dev_config_restore(dev, &dev_info, port_id) {
    eth_dev_mac_restore(dev, dev_info) {
      (*dev->dev_ops->mac_addr_set)(dev, addr); // calling virtio_mac_addr_set()
    }
  }
  ...
}

vstatic int
virtio_mac_addr_set(struct rte_eth_dev *dev, struct rte_ether_addr *mac_addr)
{
  virtio_send_command(hw->cvq, &ctrl, &len, 1);
  ...
}
```

Control channel and vring structures are the same, and they have the same init procedure.
Code in Qemu is below.
```
virtio_net_vhost_status() {
  int cvq = virtio_vdev_has_feature(vdev, VIRTIO_NET_F_CTRL_VQ) ?
                n->max_ncs - n->max_queue_pairs : 0;
  vhost_net_start(vdev, n->nic->ncs, queue_pairs, cvq) {
    int total_notifiers = data_queue_pairs * 2 + cvq;
    int nvhosts = data_queue_pairs + cvq;
    /* vrings and cvq have the same init procedure. */
    for (i = 0; i < nvhosts; i++) {
        net = get_vhost_net(peer);
        vhost_net_set_vq_index(net, i * 2, index_end);
        ...
        /* vhost protocol */
        vhost_net_start_one(get_vhost_net(peer), dev);
        if (peer->vring_enable)
            r = vhost_set_vring_enable(peer, peer->vring_enable);
     }
  }
}
```
## 2. Migration
Log dirty pages in vhost-user. Needed features:
- Vhost feature: VHOST_F_LOG_ALL (26)
- Vhost feature: VHOST_USER_PROTOCOL_F_LOG_SHMFD

As shown above, migration function **```does not```** need VirtIO feature enablement.

### Log Dirty Page Workflow
1. Qemu sends message VHOST_USER_SET_LOG_BASE with the log memory fd.
2. vhostuser vhost_user_set_log_base(): mmap(.., MAP_SHARED, fd, ..) and it stores dirty page addresses in the memory.
3. vhostuser data-path: vhost_log_cache_xx() stores used VM pages in the mapped address in #2.
4. Qemu sends message VHOST_USER_SET_LOG_FD and vhostuser closes the log memory fd.

### VHOST_F_LOG_ALL
During live migration,
the front-end may need to track the modifications the back-end makes to the memory mapped regions.
The front-end should mark the dirty pages in a log.
Once it complies to this logging,
it may declare the VHOST_F_LOG_ALL vhost feature.

With enabling it,
vhostuser will log addresses for the pages that are modified.
The address type depends on another feature VIRTIO_F_IOMMU_PLATFORM.

DPDK vhost-user **```VIRTIO_NET_SUPPORTED_FEATURES```**:
- VHOST_F_LOG_ALL and VIRTIO_F_IOMMU_PLATFORM are default in it.

### VHOST_USER_PROTOCOL_F_LOG_SHMFD
The log memory fd is provided in the ancillary data of VHOST_USER_SET_LOG_BASE message when the back-end has VHOST_USER_PROTOCOL_F_LOG_SHMFD protocol feature.

DPDK **```VHOST_USER_PROTOCOL_FEATURES```**:
- VHOST_USER_PROTOCOL_F_LOG_SHMFD is default in it.

## 3. IOMMU
```
Memory access

The front-end sends a list of vhost memory regions to the back-end using the VHOST_USER_SET_MEM_TABLE message. Each region has two base addresses: a guest address and a user address (GVA).

Messages contain guest addresses and/or user addresses to reference locations within the shared memory. The mapping of these addresses works as follows.

User addresses map to the vhost memory region containing that user address.

When the VIRTIO_F_IOMMU_PLATFORM feature has not been negotiated:
  Guest addresses (GPA) map to the vhost memory region containing that guest address.

When the VIRTIO_F_IOMMU_PLATFORM feature has been negotiated:
  Guest addresses are also called I/O virtual addresses (IOVAs). They are translated to user addresses via the IOTLB.
  The vhost memory region guest address is not used.
```
When the VIRTIO_F_IOMMU_PLATFORM feature has been negotiated,
the front-end sends IOTLB entries update & invalidation by sending VHOST_USER_IOTLB_MSG requests to the back-end with a struct vhost_iotlb_msg as payload.

## 4. Interrupt
By default, we don't need interrupt and kick in data-path, as used->idx and avail->idx are enough for polling based notification.

But the driver and the device can also enable kick and interrupt,
and there are two flags and one feature related.
| Flag/Feature | Explain | Implementation | Status |
|:---:|:---:|:---:|:---:|
| VRING_USED_F_NO_NOTIFY | device notifies driver not kicking | used->flags = VRING_USED_F_NO_NOTIFY | Flag, not a feature |
| VRING_AVAIL_F_NO_NOTIFY | driver notifies device not interrupting | avail->flags = VRING_AVAIL_F_NO_NOTIFY | Flag, not a feature |
| VIRTIO_RING_F_EVENT_IDX | device-to-driver **interrupt suppression**, i.e., reduce interrupt number | See below | Default feature in vhost-user |

### VIRTIO_VRING_F_EVENT_IDX Feature
-  **`used_event`**: the value is set by **`driver`** in the last element of available ring.
-  If **`used->idx >= used_event`**, the device must send interrupt to the driver of notifying used buffer usage.

## Reference
- vhost protocol: https://qemu.readthedocs.io/en/latest/interop/vhost-user.html
