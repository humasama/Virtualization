# Qemu Interface
This post talks about what Qemu does when creats the backend.

## Qemu Call Stack
```
// hw/net/virtio-net.c
1. virtio_net_set_link_status() -> virtio_net_set_status() -> virtio_net_vhost_status() -> calling vhost_net_start

// hw/net/vhost_net.c
2. vhost_net_start() -> vhost_net_start_one() -> calling vhost_dev_start

// hw/virtio/vhost.c
3. vhost_dev_start(): call into backend dependent callback implementations
```

Backend callback implementations for vhost-net are below.
```
// hw/virtio/vhost-backend.c
const VhostOps kernel_ops = {
        .backend_type = VHOST_BACKEND_TYPE_KERNEL,
        .vhost_backend_init = vhost_kernel_init,
        .vhost_backend_cleanup = vhost_kernel_cleanup,
        .vhost_net_set_backend = vhost_kernel_net_set_backend,
        .vhost_set_log_base = vhost_kernel_set_log_base,
        .vhost_set_mem_table = vhost_kernel_set_mem_table,
        .vhost_set_vring_addr = vhost_kernel_set_vring_addr,
        .vhost_set_vring_endian = vhost_kernel_set_vring_endian,
        .vhost_set_vring_num = vhost_kernel_set_vring_num,
        .vhost_set_vring_base = vhost_kernel_set_vring_base,
        .vhost_get_vring_base = vhost_kernel_get_vring_base,
        .vhost_set_vring_kick = vhost_kernel_set_vring_kick,
        .vhost_set_vring_call = vhost_kernel_set_vring_call,
        .vhost_set_features = vhost_kernel_set_features,
        .vhost_get_features = vhost_kernel_get_features,
        .vhost_set_backend_cap = vhost_kernel_set_backend_cap,
        .vhost_set_owner = vhost_kernel_set_owner,
        .vhost_reset_device = vhost_kernel_reset_device,
        .vhost_get_vq_index = vhost_kernel_get_vq_index,
        .vhost_set_iotlb_callback = vhost_kernel_set_iotlb_callback,
        .vhost_send_device_iotlb_msg = vhost_kernel_send_device_iotlb_msg,
        ...
};
```

Backend callback implementations for vhost-user are below.
```
// hw/virtio/vhost-user.c
const VhostOps user_ops = {
        .backend_type = VHOST_BACKEND_TYPE_USER,
        .vhost_backend_init = vhost_user_backend_init,
        .vhost_backend_cleanup = vhost_user_backend_cleanup,
        .vhost_backend_memslots_limit = vhost_user_memslots_limit,
        .vhost_set_log_base = vhost_user_set_log_base,
        .vhost_set_mem_table = vhost_user_set_mem_table,
        .vhost_set_vring_addr = vhost_user_set_vring_addr,
        .vhost_set_vring_endian = vhost_user_set_vring_endian,
        .vhost_set_vring_num = vhost_user_set_vring_num,
        .vhost_set_vring_base = vhost_user_set_vring_base,
        .vhost_get_vring_base = vhost_user_get_vring_base,
        .vhost_set_vring_kick = vhost_user_set_vring_kick,
        .vhost_set_vring_call = vhost_user_set_vring_call,
        .vhost_set_features = vhost_user_set_features,
        .vhost_get_features = vhost_user_get_features,
        .vhost_set_owner = vhost_user_set_owner,
        .vhost_reset_device = vhost_user_reset_device,
        .vhost_get_vq_index = vhost_user_get_vq_index,
        .vhost_set_vring_enable = vhost_user_set_vring_enable,
        .vhost_requires_shm_log = vhost_user_requires_shm_log,
        .vhost_migration_done = vhost_user_migration_done,
        .vhost_backend_can_merge = vhost_user_can_merge,
        .vhost_net_set_mtu = vhost_user_net_set_mtu,
        .vhost_set_iotlb_callback = vhost_user_set_iotlb_callback,
        .vhost_send_device_iotlb_msg = vhost_user_send_device_iotlb_msg,
        .vhost_get_config = vhost_user_get_config,
        .vhost_set_config = vhost_user_set_config,
        ...
};
```
## vhost_dev_start()
**```vhost_dev_start()```** implements vhost protocol and it calls backend depenent implementations
to support different backent types.

```
vhost_dev_set_features(hdev, hdev->log_enabled);

hdev->vhost_ops->vhost_set_mem_table(hdev, hdev->mem);

for (i = 0; i < hdev->nvqs; ++i) {
  vhost_virtqueue_start(hdev, vdev, hdev->vqs + i, hdev->vq_index + i);
}

/* hdev->log_enabled is set to false by default in vhost_dev_init() which instances the backend when Qemu starts */
/* If VHOST_F_LOG_ALL is negotiated, it is set to true (vhost_log_global_start()) */
if (hdev->log_enabled) {
  uint64_t log_base;
  hdev->log_size = vhost_get_log_size(hdev);
  hdev->log = vhost_log_get(hdev->log_size,
                            vhost_dev_log_is_shared(hdev));
  log_base = (uintptr_t)hdev->log->log;
  hdev->vhost_ops->vhost_set_log_base(hdev, hdev->log_size ? log_base : 0, hdev->log);
}

if (hdev->vhost_ops->vhost_dev_start) {
  hdev->vhost_ops->vhost_dev_start(hdev, true);
}

if (vhost_dev_has_iommu(hdev) &&
    hdev->vhost_ops->vhost_set_iotlb_callback) {
        hdev->vhost_ops->vhost_set_iotlb_callback(hdev, true);
    for (i = 0; i < hdev->nvqs; ++i) {
        struct vhost_virtqueue *vq = hdev->vqs + i;
        vhost_device_iotlb_miss(hdev, vq->used_phys, true);
    }
}
```

## Features (TODO)
1. VHOST_USER_PROTOCOL_F_LOG_SHMFD/VHOST_F_LOG_ALL

vhost_log_global_start() is in charge of handling VHOST_F_LOG_ALL feature bit.
```
vhost_dev_init()
{
  hdev->memory_listener = (MemoryListener) {
    .name = "vhost",
    .begin = vhost_begin,
    .commit = vhost_commit,
    .region_add = vhost_region_addnop,
    .region_nop = vhost_region_addnop,
    .log_start = vhost_log_start,
    .log_stop = vhost_log_stop,
    .log_sync = vhost_log_sync,
    .log_global_start = vhost_log_global_start,
    .log_global_stop = vhost_log_global_stop,
    .eventfd_add = vhost_eventfd_add,
    .eventfd_del = vhost_eventfd_del,
    .priority = 10
    };

    hdev->iommu_listener = (MemoryListener) {
      .name = "vhost-iommu",
      .region_add = vhost_iommu_region_add,
      .region_del = vhost_iommu_region_del,
    };
}
```
