# VirtIO
## Memory
The split virtqueue format separates the virtqueue into three areas, where each area is writable by either the driver or the device, but not both:
- Descriptor Area: used for describing buffers.
- Driver Area: data supplied by driver to the device. Also called avail virtqueue.
- Device Area: data supplied by device to driver. Also called used virtqueue.

They need to be allocated in the **```driver’s memory```** for it to be able to access them in a straightforward way.
Buffer **addresses are stored from the driver's point of view**, and the device needs to perform an address translation.

There are many ways for the device to access it depending on the latter nature:
- For an emulated device in the hypervisor (like qemu), the guest's address is in its own process memory.
- For other emulated devices like **```vhost-net or vhost-user```**, a memory mapping needs to be done, like **```POSIX shared memory```**. A file descriptor to that memory is shared through vhost protocol.
- For a real device a **hardware-level** translation needs to be done, usually via IOMMU.

### References
- https://www.redhat.com/en/blog/virtqueues-and-virtio-ring-how-data-travels

## VirtIO Device Configuration Workflow
When VirtIO driver configure the device,
Both **Qemu** and **vhost-net and vhost-user** do actions to set up vhost via vhost protocol

### Qemu Call Stack
When the driver configures the device,
Qemu is notified of the device status change, and
the following functions are invoked to start the vhost setup.

```
//Qemu ops to communicate with VirtIO driver.
// hw/net/virtio-net.c
static NetClientInfo net_virtio_info = {
    .type = NET_CLIENT_DRIVER_NIC,
    .size = sizeof(NICState),
    .can_receive = virtio_net_can_receive,
    .receive = virtio_net_receive,
    .link_status_changed = virtio_net_set_link_status,
    .query_rx_filter = virtio_net_query_rxfilter,
    .announce = virtio_net_announce,
};
virtio_net_set_link_status ->
  virtio_net_set_status ->
    virtio_net_vhost_status:
      cvq = virtio_vdev_has_feature(vdev, VIRTIO_NET_F_CTRL_VQ) ?
              n->max_ncs - n->max_queue_pairs : 0;
      vhost_net_start(..., queue_pairs, cvq) {
        nvhosts = data_queue_pairs + cvq;
        for i in nvhosts:
          - vhost_net_start_one() {
              -- vhost_dev_enable_notifiers ->
                  virtio_bus_set_host_notifier() ->
                    event_notifier_init -> eventfd() // QEMU registers an ioeventfd for the VIRTIO_PCI_QUEUE_NOTIFY hardware register access
              -- vhost_dev_start() {
                 |  vhost_dev_set_features;
                 |  hdev->vhost_ops->vhost_set_mem_table(hdev, hdev->mem);
                 |  vhost_virtqueue_start {
                 |    dev->vhost_ops->vhost_set_vring_addr(dev, &addr);
                 |    dev->vhost_ops->vhost_set_vring_kick(dev, &file); //send ioeventfd value to vhost
                 |    dev->vhost_ops->vhost_set_vring_call(dev, &file); //send irqfd value to vhost
                 |  }
                 |  hdev->vhost_ops->vhost_dev_start(hdev, true);
                 |  hdev->vhost_ops->vhost_set_iotlb_callback(hdev, true); //vhost_dev_has_iommu(hdev);
                 |}
              -- if (net->nc->info->type == NET_CLIENT_DRIVER_TAP) {
                 |  qemu_set_fd_handler(net->backend, NULL, NULL, NULL);
                 |  for (file.index = 0; file.index < net->dev.nvqs; ++file.index)
                 |    vhost_net_set_backend(&net->dev, &file);
                 |}
            }
          - vhost_set_vring_enable();
        }
      }
```
### 1. Vhost-net
<img src="https://github.com/humasama/technologies/assets/6119088/eede915e-1460-464d-99f0-1ba80efbfac7" width=600>

### 2. Vhost-user
<img src="https://github.com/humasama/technologies/assets/6119088/6ec6d070-3caa-4531-875a-ce27a10cdc7f" width=400>

## Notification
**Question**: How do notifications happen?

**Answer**: The datapath workflow is below:
1. The driver writes **```VIRTIO_PCI_QUEUE_NOTIFY```** hardware register (**```VIRTIO_PCI_NOTIFY_CFG base + VIRTIO_PCI_CAP_COMMON_CFG.queue_notify_off * #queue```**);
2. VM exit (trap like exception) and KVM VM-exit handler handles this kick;
3. vhost-net sends packets to the tap device;
4. After handling I/O requests, vhost-net sends a signal to the guest with **```irqfd```**.

### References
- vhost-user: https://www.redhat.com/en/blog/journey-vhost-users-realm
- vhost-net: https://www.redhat.com/en/blog/deep-dive-virtio-networking-and-vhost-net
- https://insujang.github.io/2021-03-15/virtio-and-vhost-architecture-part-2/
- http://blog.vmsplice.net/2011/09/qemu-internals-vhost-architecture.html

<img src="https://github.com/humasama/technologies/assets/6119088/4757547f-4872-4429-9ab4-8a721fab748b" width=400>
