# VirtIO PMD - Configure Device
This is the second post.
Let's talks about VirtIO PMD configuration implementation and related KVM/QEMU implementation.

## Configure Device
The workflow of VirtIO PMD configuration is below:
1. probe: **```virtio_init_device```**(eth_dev, VIRTIO_PMD_DEFAULT_GUEST_FEATURES).
2. virtio_dev_configure: negotiate features. If features change, call virtio_init_device to re-init device.
3. virtio_dev_start: SW prepare and it doesn't touch VirtIO device.

### virtio_init_device()
```
/* VIRTIO_OPS(hw)->set_status(hw, VIRTIO_CONFIG_STATUS_RESET), VM exit */
virtio_reset(hw);

/* VM exit. */
virtio_set_status(hw, VIRTIO_CONFIG_STATUS_ACK);

/* VM exit. */
virtio_set_status(hw, VIRTIO_CONFIG_STATUS_DRIVER);

virtio_alloc_queues(eth_dev) {
  for (i = 0; i < nr_vq; i++) {
    vq_size = VIRTIO_OPS(hw)->get_queue_num(hw, queue_idx);
    virtqueue_alloc(hw, queue_idx, vq_size, queue_type, numa_node, vq_name) {
      if (hw->use_va) {
        vq->vq_ring_mem = (uintptr_t)mz->addr;
        vq->mbuf_addr_offset = offsetof(struct rte_mbuf, buf_addr);
      } else {
        /* Using DPDK in guest with no-iommu uses PA as IOVA */
        vq->vq_ring_mem = mz->iova;
        vq->mbuf_addr_offset = offsetof(struct rte_mbuf, buf_iova);
      }
      /* Init split/packed vring structures (SW operation) */
      virtio_init_vring(vq);
    }
    /* Store desc/avail/used ring addresses to the device and VM exit. */
    VIRTIO_OPS(hw)->setup_queue(hw, vq);
  }
}

/* Qemu is notified and starts to set up vhost (either vhost-user or vhost-net) */
virtio_reinit_complete(hw) {
  virtio_set_status(hw, VIRTIO_CONFIG_STATUS_DRIVER_OK);
}
```
After setting device status to **```VIRTIO_CONFIG_STATUS_DRIVER_OK```**,
Qemu starts to set up vhost via vhost protocol.
DRIVER_OK status means VirtIO driver's work is done and QEMU starts to work.

## QEMU Operation due to Device Setting
As shown in the code above,
setting VirtIO device registers, like ```virtio_reset(hw)```, causes VM Exit,
and QEMU finally handles the exit reason.
To achieve this workflow, what does HW provide, KVM do, and QEMU do?
See https://github.com/humasama/technologies/blob/main/Kernel/KVM_QEMU.md.
