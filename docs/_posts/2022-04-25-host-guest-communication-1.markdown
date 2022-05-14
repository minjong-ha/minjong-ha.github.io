---
layout: posts
title:  "Host-Guest Communication: Full vs Para Virtualization - 1"
author: Minjong Ha
published: false
date:   2022-04-25 17:13:43 +0900
---

About Host-Guest communication!
Currently writing content....

In this post, I share my research and analysis about the data communication between the host and guest in case of the device request and focus on the differences in the full and para virtualized machine.
I also explain about the vCPU as a background; what is the vCPU and how it works.


## Introduction

A virtual machine in "Full-Virtualization" does not know that it is operating in a virtualized environment.
On the other hand, a VM in "Para-Virtualization" knows that it is since the hypervisor who emulates the device notifies it.
Since the VM already knows that the device is for the virtualization, it has different device request logic.
The para virtualization reduces the data copy between the user and the kernel compared to full virtualization which is one of the most largest overhead in the Host-Guest communication

One of the most famous para-virtualization is "Virtio".
"Virtio" presents the devices and drivers specially dedicated for the virtualization.
If the user specifies the virtio device to the hypervisor, VM tries to connect to the device with virtio driver.
Since the virtio driver is installed in Linux as default, it can be activated through kernel config.
However, in Windows, it is not included in the Windows kernel and the user should install virtio driver manually in the VM.

For example, you can configurate your disk device to the normal SATA emulation or the virtio device disk.
In my personal experiments, virtio outperforms the SATA emulation approximately 30-50% in the sysbench - FIO test.


## Background

### ioctl()

The ioctl() system call manipulates the underlying device parameters of special files([reference](https://man7.org/linux/man-pages/man2/ioctl.2.html)).
Usually, it is used to configurate and send requests to the device.
For instance, user could configurate the printer device via ioctl(); such as what is the font it using and what is the size of the paper in the device.
We can see the information about the ioctl()s that KVM supports in __qemu/linux-headers/linux/kVM.h__


### vCPU()

QEMU-KVM hypervisor supports vCPU, executing the code in the guest directly on the physical CPU.
When the QEMU-KVM hypervisor emulates the CPU for the VM, the emulated CPU is supported by vCPU in the KVM.
When the vCPU enters to the GUEST_MODE, the guest can use it exclusively.
<!-- Otherwise, it should runs operations that incur large overhead, such as code generation for instructions through the QEMU. -->
If there were no KVM support, which means in the QEMU only hypervisor, every guest code should be handled by the QEMU.
QEMU translates the guest code to the suitable instructions for the physical CPUs in the host and the cost is tremendous.
However ,thanks to vCPU, the QEMU-KVM hypervisor leaverages the overall performance of the VM.

Then how the VM requests and initializes vCPU from the host(KVM)?
When the hypervisor starts to emulate virtual devices, it also emulates the virtual CPUs for the VM.

```c
static const TypeInfo kvm_accel_ops_type = {
  ...
  .class_init = kvm_accel_ops_class_init,
  ...
}

static void kvm_accel_ops_class_init(ObjectClass *oc, void *data) {
  AccelOpsClass *ops = ACCEL_OPS_CLASS(oc);

  **ops->create_vcpu_thread = kvm_start_vcpu_thread**
  ...
}
```

The above codes are in qemu/accel/kvm/kvm-accel-ops.c.
The QEMU defines the kvm_accel_ops_type, and allocates the callback functions for the vCPU.
Thus, when the QEMU tries to allocate vCPU, the kvm_start_vcpu_thread() funcion will be called.

```c
static void x86_cpu_realizefn(DeviceState *dev, Error **errp) {
  ...
  CPUState *cs = CPU(dev);
  x86CPU *cpu = X86_CPU(dev);
  x86CPUClass *xcc = X86_CPU_GET_CLASS(Dev);
  CPUX86State *env = &cpu->env;
  ...
  **qemu_init_vcpu(cs);**
}
```

```c
void qemu_init_vcpu(CPUState *cpu) {
  ...
**cpus_accel->create_vcpu_thread(cpu);**
  ...
}
```

The above codes are in qemu/target/i386/cpu.c and qemu/softmmu/cpus.c.
The hypervisor realize the virtual CPUs via x86_cpu_realizefn() and it calls qemu_init_vcpu().
In this function, it starts to create the threads for the vCPU.


```c
static void kvm_start_vcpu_thread(CPUState *cpu) {
  char thread_name[VCPU_THREAD_NAME_SIZE]; //#define VCPU_THREAD_NAME_SIZE 16

  cpu->thread = g_malloc0(sizeof(QemuThread)); //alloc cpu thread memory
  cpu->halt_cond = g_malloc0(sizeof(QemuCond));
  ...
  **qemu_thread_create(cpu->thread, thread_name, kvm_vcpu_thread_fn, cpu, QEMU_THREAD_JOINABLE);**
}
```

```c
void qemu_thread_create(QemuThread *thread, const char *name, void *(*start_routine)(void *), void *arg, int mode) {
  ...
  QemuThreadArgs *qemu_thread_args;
  ...
  qemu_thread_args = g_new0(QemuThreadsArgs, 1);
  qemu_thread_args->name = g_strdup(name);
  qemu_thread_args->start_routine = start_routine; //start_routine == kvm_vcpu_thread_fn
  qemu_trehad_args->arg = arg

  **err = pthread_create(&thread->thread, &attr, qemu_thread_start, qemu_thread_args);**
  ...
}

static void *qemu_thread_start(void *args) {
  ...
  void *(*start_routine)(void *) = qemu_thread_args->start_routine; //start_routine == kvm_vcpu_thread_fn
  ...
  pthread_cleanup_push(qemu_thread_atexit_notify, NULL);
  **r = start_routine(arg);** //start_routine == **kvm_vcpu_thread_fn**
  pthread_cleanup_pop(1);
  ...
}
```

The above codes are in qemu/accel/kvm/kvm_accel-ops.c and qemu/util/qemu-thread-posix.c
Now, we can understand that the vCPU is the posix thread actually, and this thread runs kvm_vcpu_thread_fn() function.

```c
static void *kvm_vcpu_thread_fn(void *arg) {
  CPUState *cpu = arg;
  ...
  qemu_thread_get_self(cpu->thread);
  cpu->thread_id = qemu_get_thread_id();
  ...
  current_cpu = cpu;

  **r = kvm_init_vcpu(cpu, &error_fatal);**
  kvm_init_cpu_signals(cpu);

  cpu_thread_signal_created(cpu); // cpu->created = true;
  ...
  do {
    if (cpu_can_run(cpu)) {
      **r = kvm_cpu_exec(cpu);**
      ...
    }
  } while (!cpu->unplug || cpu_can_run(cpu));
  ...
}
```

In vCPU thread, kvm_vcpu_thread_fn(), QEMU requests to allocate the vCPUs to the KVM via ioctl.
qemu_geut_thread_id() calls ioctl() and the KVM returns the vCPU fd.
After the QEMU receives the vCPU, now it executes it with kvm_cpu_exec().
Since we are in focusing on how the vCPU actually works, I will explain how the vCPUs are created by KVM in ohter posts later.

```c
int kvm_cpu_exec(CPUState *cpu) {
  struct kvm_run *run = cpu->kvm_run;
  ...
  do {
    ...
    kvm_arch_put_registers(cpu, KVM_PUT_RUNTIME_STATE);
    ...z
    **run_ret = kvm_vcpu_ioctl(cpu, KVM_RUN, 0);**
    ...
    switch (**run->exit_reason**) {
      ...
      case KVM_EXIT_MMIO:
        DPRINTF("handle_mmio\n");
        address_space_rw(&address_space_memory,
                         run->mmio.phys_addr, attrs,
                         run->mmio.data,
                         run->mmio.len,
                         run->mmio.is_write);
        ret = 0;
      break;
      ...
      default:
        DPRINTF("kvm_arch_handle_exit\n");
        ret = kvm_arch_handle_exit(cpu, run);
        break;
  } while (ret == 0);
  
  cpu_exec_end(cpu);
  ...
}

int kvm_vcpu_ioctl(CPUState *cpu, int type, ...) {
  ...
  ret = ioctl(cpu->kvm_fd, type, arg); //type == KVM_RUN
  ...
}
```

In kvm_cpu_exec(), we can check that it has two parts: one is the 'kvm_vcpu_ioctl()' and another is the 'switch(run->exit_reason)'.
'kvm_vcpu_ioctl()' is the point makes vCPU switching to the GUEST_MODE.
QEMU notifies that the vCPU is in ready to run in GUEST_MODE via ioctl().
KVM_RUN flag represents how the KVM handles this ioctl() requests.
In the contents below, I will explain how the KVM makes the vCPU, the physical CPU that runs the vCPU thread, into the GUEST_MODE.
But for now, lets focus on the part after the ioctl().

The KVM returns the result of the kvm_vcpu_ioctl() when the VM_EXIT happens.
The VM_EXIT makes the CPU to escape from the GUEST_MODE and returns it to the so called HOST_MODE; If the CPU is in the Intel architecture, they are called non-root mode (GUEST_MODE), and root mode (HOST_MODE).
Now for better understanding, lets assume that there is a machine having only one physical CPU, and it tries to run the VM.






## Full-Virtualization


## Para-Vitualization

## Experiment

## Conclusion


[Notion Document: Full-Virtualization(QEMU-KVM) vs Para-Virtualization(Virtio)](https://seen-fact-e72.notion.site/Full-Virtualization-vs-Para-Virtualization-cd4933792f6a4a2b871a385f58592955)
