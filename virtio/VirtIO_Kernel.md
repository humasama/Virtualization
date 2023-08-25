# VirtIO Kernel

code files:
- drivers/virtio/*
- drivers/net/virtio_net.c

## Kernel Procedure
1. Kernel probes **virtio_net.ko** that registers **virtio_net_driver** to **VirtIO Bus**;
2. Kernel scans PCIe bus and adds found VirtIO devices to VirtIO Bus via **virtio_pci_driver**;
3. Kernel scans VirtIO Bus;
4. Kernel calls registered drivers (e.g., **virtio_net_driver**) for devices in the VirtIO Bus.

### virtio_pci_driver
virtio_pci_driver is not a module, but VirtIO PCI device driver.
Kernel uses it to probe the device when scans a VirtIO PCI device.
In probe(), **```register_virtio_device()```** is called to register the device to VirtIO Bus.

```
// drivers/virtio/virtio_pci_common.c
static struct pci_driver virtio_pci_driver = {
	.name		= "virtio-pci",
	.id_table	= virtio_pci_id_table,
	.probe		= virtio_pci_probe,
	.remove		= virtio_pci_remove,
#ifdef CONFIG_PM_SLEEP
	.driver.pm	= &virtio_pci_pm_ops,
#endif
	.sriov_configure = virtio_pci_sriov_configure,
};
module_pci_driver(virtio_pci_driver);

static int virtio_pci_probe(struct pci_dev *pci_dev,
			    const struct pci_device_id *id) {
  ...
  register_virtio_device(&vp_dev->vdev);
  ...
}
```

## VirtIO Bus
**```register_virtio_device()```**
and
**```register_virtio_driver()```** are used to add device and its driver to VirtIO Bus.

```
// drivers/virtio/virtio.c
static int virtio_init(void)
{
	if (bus_register(&virtio_bus) != 0)
		panic("virtio bus registration failed");
	return 0;
}

static void __exit virtio_exit(void)
{
	bus_unregister(&virtio_bus);
	ida_destroy(&virtio_index_ida);
}
core_initcall(virtio_init);
module_exit(virtio_exit);
```

## virtio_net.ko (virtio_net_driver)
- virtio_net.ko is the VirtIO network device driver.
- Kernel calls **```virtio_net_driver_init```** aotumatically as virtio_net.ko is a built-in kernel module.
- virtio_net_driver_init() registers the **driver** *register_virtio_driver* into VirtIO Bus.

Note that, virtio_net.ko only registers the DRIVER to VirtIO Bus.
When scans VirtIO Bus,
it will be used to probe VirtIO network devices.

```
// drivers/net/virtio_net.c
static __init int virtio_net_driver_init(void)
{
  ...
  register_virtio_driver(&virtio_net_driver);
  ...
}
module_init(virtio_net_driver_init);
```
```
static struct virtio_driver virtio_net_driver = {
	.feature_table = features,
	.feature_table_size = ARRAY_SIZE(features),
	.feature_table_legacy = features_legacy,
	.feature_table_size_legacy = ARRAY_SIZE(features_legacy),
	.driver.name =	KBUILD_MODNAME,
	.driver.owner =	THIS_MODULE,
	.id_table =	id_table,
	.validate =	virtnet_validate,
	.probe =	virtnet_probe,
	.remove =	virtnet_remove,
	.config_changed = virtnet_config_changed,
  ...
};
```
