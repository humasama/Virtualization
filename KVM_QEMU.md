# KVM/QEMU Virtual Machine

## QEMU Workflow
The below figure illustrates QEMU call stack to run a VM.
- VM code runs by a pthread created by QEMU.
- For a VM, its core and memory are provided by host KVM;
- For a VM, QEMU emulates devices and handles I/O related VM Exit;
- KVM sub-system is represented as a device and used by ioctl();
- QEMU explicitly calls KVM ioctl() to create/handle VM Exit/destroy VM.

![image](https://github.com/humasama/technologies/assets/6119088/6c61275b-7c0b-4bd6-a87e-95cdbb101bd0)

## KVM Workflow
KVM represented itself as **```THREE character devices```**,
User-space hypervisors, QEMU, uses its features via **```IOCTL```**.

### kvm_chardev Device
This character device is in charge of creating VM (**```KVM_CREATE_VM```**).

<img width="700" alt="image" src="https://github.com/humasama/technologies/assets/6119088/cf4be0e9-7877-44e8-b909-5608069259d2">

### kvm_vm Device
This character device is in charge of creating vCPU (**```KVM_CREATE_VCPU```**) and memory (**```KVM_SET_USER_MEMORY_REGION```**).

<img width="700" alt="image" src="https://github.com/humasama/technologies/assets/6119088/40d030d2-bc3f-4603-be5d-91b1902429b9">

### kvm_vcpu Device
This character device is in charge of running VM (**```KVM_RUN```**),
like handing VM exit events.
QEMU uses it to handle all events for VM during VM runtime.

<img width="500" alt="image" src="https://github.com/humasama/technologies/assets/6119088/1a9dec95-e432-482f-a112-77662078515e">

## Reference
- KVM: https://lwn.net/Articles/658511/
