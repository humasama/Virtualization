# Linux Devices
All hardware devices look like regular files;
they can be opened, closed, read and written using the same, standard,
system calls that are used to manipulate files.
**```Every device```** in the system is **```represented```** by a **```device special file```**.

Devices are accessed by the user through special device files,
These files are grouped into the /dev directory, and system calls open, read, write, close, lseek, mmap etc.
are redirected by the operating system to the device driver associated with the physical device.
The device driver is a kernel component (usually a module) that interacts with a hardware device.

Linux supports three types of hardware device: **```character```**, **```block```** and **```network```**.

## Device File
For block and character devices,
these device special files are created by the **```mknod```** command and they describe the device using major and minor device numbers.
Network devices are also represented by device special files but they are created by Linux as it finds and initializes the network controllers in the system.

- character: slow devices. System calls directly go to device drivers. A character device can be a real HW device, like Intel DSA, or a virtual device, like tap/tun/vhost-net.
- block: data volume is large and search is common. System calls go to the file management subsystem and the block device subsystem.

### Network Device File
For different network devices, their device file layouts are standardized.
In a device file, there are 8 address spaces and each max size is **```0x10000000000 (2^40B)```**:
- the first 6 address spaces are used for device memory spaces located by 6 BARs;
- the 7th address space is for ROM;
- the 8th address space is PCI configuration space.

Note that the **```device file layout is standardized```**.
Specifically, the offset of each address space within the device file is calculated by **```index<<40```**.

## Character Device Driver
The character device drivers receive unaltered system calls made by users over device-type files.
Consequently, implementation of a character device driver means implementing the system calls specific to files: open, close, read, write, lseek, mmap, etc.
These operations are described in the fields of the struct file_operations structure.
```
#include <linux/fs.h>
struct file_operations {
    struct module *owner;
    loff_t (*llseek) (struct file *, loff_t, int);
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    [...]
    long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
    [...]
    int (*open) (struct inode *, struct file *);
    int (*flush) (struct file *, fl_owner_t id);
    int (*release) (struct inode *, struct file *);
    [...]
```
Note that besides read/write operations, a driver needs the ability to perform certain physical device control tasks.
These operations are accomplished by implementing a **```ioctl```** function.
### Registeration/Unregistration of Character Devices
The registeration and unregistration of a character device is done by calling functions in driver below.
```
#include <linux/fs.h>

int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name);
int register_chrdev_region(dev_t first, unsigned int count, char *name);
void unregister_chrdev_region(dev_t first, unsigned int count);
```
### Character Deivce: Vhost-net
Linux vhost-net is a miscellaneous driver,
and a miscellaneous driver is a special and simple character device driver.

```
static struct miscdevice vhost_net_misc = {
	.minor = VHOST_NET_MINOR,
	.name = "vhost-net",
	.fops = &vhost_net_fops,
};

static int vhost_net_init(void)
{
	if (experimental_zcopytx)
		vhost_net_enable_zcopy(VHOST_NET_VQ_TX);
	return misc_register(&vhost_net_misc);
}
module_init(vhost_net_init);
```
### Character Deivce: TAP/TUN
```
static const struct file_operations tap_fops = {
	.owner		= THIS_MODULE,
	.open		= tap_open,
	.release	= tap_release,
	.read_iter	= tap_read_iter,
	.write_iter	= tap_write_iter,
	.poll		= tap_poll,
	.llseek		= no_llseek,
	.unlocked_ioctl	= tap_ioctl,
	.compat_ioctl	= compat_ptr_ioctl,
};
```

## References
- https://linux-kernel-labs.github.io/refs/heads/master/labs/device_drivers.html
- https://tldp.org/LDP/tlk/dd/drivers.html
- Linux book: https://lwn.net/Kernel/LDD3/
