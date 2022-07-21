---
layout: posts
title:  "Host-Guest Communication: Full vs Para Virtualization - 1"
author: Minjong Ha
published: true
date:   2022-04-25 17:13:43 +0900
---

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

Now lets discuss about how the VM requests and initialize vCPU from the KVM.
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

In kvm_cpu_exec(), we can check that it consists of the two parts: one is the 'kvm_vcpu_ioctl()' and another is the 'switch(run->exit_reason)'.
'kvm_vcpu_ioctl()' is the point makes vCPU switching to the GUEST_MODE.
QEMU notifies that the vCPU is in ready to run in GUEST_MODE via ioctl().
KVM_RUN flag represents how the KVM handles the ioctl() requests.
In the contents below, I will explain how the KVM makes the vCPU, the physical CPU that runs the vCPU thread, into the GUEST_MODE.
But for now, lets focus on the part after the ioctl().

The KVM returns the result of the kvm_vcpu_ioctl() when the VM_EXIT happens.
The VM_EXIT makes the CPU to escape from the GUEST_MODE and returns it to the so called HOST_MODE; If the CPU is in the Intel architecture, they are called non-root mode (GUEST_MODE), and root mode (HOST_MODE).
Usually, there are two reasons what trigger the VM_EXIT: to handle the request that cannot be done in the GUEST_MODE, and the timer expire.
For better understanding, assumes that there is a machine having only one physical CPU, and it tries to run the VM with QEMU-KVM hypervisor.
First I will explain about the timer reasoned VM_EXIT.
Since the machine has only one physical CPU, it cannot proceed any host's task if the CPU is in the GUEST_MODE.
The CPU in the GUEST_MODE is only for the virtualized system. 
Thus, without the periodical VM_EXIT, the machine could not comeback from the GUEST_MODE and every host's tasks wait over and over.
To prevent this disaster, every CPUs wake up from the GUEST_MODE in every configurated timeslices.
It also means if we want to give more execution time to the VM, we could achieve it via extending the timeslices for the GUEST_MODE.
Another reason is the operations that cannot be done in the GUEST_MODE.
For example, the requests for the emulated devices usually trigger the VM_EXIT to complete actions.
The QEMU hypervisor emulates the virtual devices for the VM.
With the support of the BIOS for the VM, It could interact these virtual, logical devices.
The Guest inside the VM hands over the requests that executed inside the GUEST to the virtual devices.
However, they are not the real, physical devices.
If the VM tries to send network packet to the other, the data should pass the physical network device.
Since the emulated devices is not a real device, it cannot be done in the GUEST_MODE.
Only the host can complete the action by passing over the data from the emulated device to the real device.
If there is a writes(requests) on the memory area that controlled by the emulated devices, it triggers the VM_EXIT and the CPU wakes up from the GUEST_MODE.
Now, the CPU can proceed the host tasks.
The KVM that wakes up from the GUEST_MODE, tries to figure out the reason of the VM_EXIT and returns the reason to the QEMU.
And this is the point where the 'switch(run->exit_reason)' works.
QEMU completes the requests depending on the 'exit_reason'.
And after the QEMU finishes the operations, it calls kvm_cpu_exec() again since it is performed in do while loop.

Now lets move on to the most important part, where the KVM actually runs vCPU.

```c
int kvm_cpu_exec(CPUState *cpu) {
  struct kvm_run *run = cpu->kvm_run;
  ...
  do {
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

As we already checked above, kvm_vcpu_ioctl() calls ioctl() with KVM_RUN flag.


```c
static long kvm_vcpu_ioctl(struct file, *filp, unsigned int ioctl, unsigned long arg) {
  struct kvm_vcpu *vcpu = filp->private_data;
  ...
  switch (ioctl) {
  case KVM_RUN:
    ...
    **r = kvm_arch_vcpu_ioctl_run(vcpu);**
  }
  ...
}
```

```c
int kvm_arch_vcpu_ioctl_run(struct kvm_vcpu *vcpu) {
  struct kvm_run *kvm_run = vcpu->run;
  ...
  r = vcpu_run(vcpu);
  ...
}

static int vcpu_run(struct kvm_vcpu *vcpu) {
  struct kvm *kvm = vcpu->kvm;
  ..
  for(;;) {
    if (kvm_vcpu_running(vcpu)
      **r = vcpu_enter_guest(vcpu);**
  ...
  }
  ...
}

static int vcpu_enter_guest(struct kvm_vcp *vcpu) {
  ...
  vcpu->mode = IN_GUEST_MODE;
  ...
  exit_fastpath = static_call(kvm_x86_run)(vcpu);
  ...
  for (;;) {
    ...
    **exit_fastpath = static_call(kvm_x86_run)(vcpu); 
    if(likely(exit_fastpath != EXIT_FASTPATH_REENTER_GUEST))
      break;
    ...
    if(unlikely(kvm_vcpu_exit_request(vcpu))) {
      exit_fasthpath = EXIT_FASTPATH_EXIT_HANDLED;
      break;
    }**
  }
  ...
  **vcpu->mode = OUTSIDE_GUEST_MODE;**
  ...
  **r = static_call(kvm_x86_handle_exit)(vcpu, exit_fastpath);**
}
```

Above codes are represent the flow of the KVM ioctl handling.
KVM handles ioctl() for the KVM with the KVM_RUN through __kvm_arch_vcpu_ioctl_run()__.
It leads the KVM to the __vcpu_enter_guest()__, which is the most important code I think.
In the for loop in it,  __static_call(kvm_x86_run)(vcpu)__ is the point where the CPU switches itself into the GUEST_MODE.


<!-- need screenshots -->
<img data-action="zoom" src='{{ "../assets/images/2022-04-25-host-guest-communication/vmx_vcpu_run-1.png" | relative_url }}' alt='relative'>

It calls an assembly function in the image.
Since my machine has Intel CPU, vmx_vcpu_run() is called (It is called smx under the AMD CPU).
In this function, we can see that the cpu tries to load and save the cpu's state data.
I estimate this is the part where the VMCS (Virtual Machine Control Structure) switch happens, but it is not sure.
If my assume is right, this is the part where the machine prepare the HOST-GUEST mode switch.

<img data-action="zoom" src='{{ "../assets/images/2022-04-25-host-guest-communication/vmx_vcpu_run-2.png" | relative_url }}' alt='relative'>

After it saves all HOST's state data and load GUEST's state data on the CPU, it calls vmenter() function

<img data-action="zoom" src='{{ "../assets/images/2022-04-25-host-guest-communication/vmx_vmenter.png" | relative_url }}' alt='relative'>

This is the instruction that makes CPU mode into the GUEST_MODE in the vmenter() function through vm_resume, and vm_launch.


```c
static int handle_io(struct kvm_vcpu *vcpu) {
  ...
  if (string)
    **return kvm_emulate_instruction(vcpu, 0);**
    ...
  return kvm_fast_pio(vcpu, size, port, in);
}
```

```c
int kvm_emulate_instruction(struct kvm_vcpu *vcpu, int emulation_type) {
  return x86_emulate_instruction(vcpu, 0, emulation_type, NULL, 0);
}

int x86_emulate_instruction(struct kvm_vcpu *vcpu, gpa_t cr2_or_gpa, int emulation_type, void *insn, int insn_len) {
  ...
  else if(vcpu->mmio_needed) {
    ...
    **vcpu->arch.complete_userspace_io = complete_emulated_mmio;**
  }
  ...
}

static int complete_emulated_mmio(struct kvm_vcpu *vcpu) {
  struct kvm_run *run = vcpu->run;
  struct kvm_mmio_fragment *frag;
  ...
  frag = &vcpu->mmio_fragments[vcpu->mmio_cur_fragment];
  ...
  if(!vcpu->mmio_is_write)
    memcpy(frag->data, run->mmio.data, len);
  ...
  run->exit_reason = KVM_EXIT_MMIO;
  run->mmio.phys_addr = frag->gpa;
  if (vcpu->mmio_is_write)
    memcpy(run->mmio.data, frag->data, min(8u, frag->len));
  ...
}
```

Above codes are the one of the exit handling by KVM: the device MMIO request.
After the KVM completes the works it should do, it returns the control to the QEMU.

<img data-action="zoom" src='{{ "../assets/images/2022-04-25-host-guest-communication/overall_flow.png" | relative_url }}' alt='relative'>

The image represents the overall code flow of the vCPU execution.
We saw that the vCPU enters to the GUEST_MODE and exits periodically with the detail codes.
The vCPU posix threads on the HOST exist only to secure the running time of the GUEST_MODE through the scheduling.

Now we understand how the vCPU works in QEMU-KVM hypervisor and ready to compare the difference between the Full-Virtualization and Para-Virtualization.
I will explain about it in the next post.



<!---

## Full-Virtualization


## Para-Vitualization

## Experiment

## Conclusion
--->


[Notion Document: Full-Virtualization(QEMU-KVM) vs Para-Virtualization(Virtio) (written in Korean)](https://seen-fact-e72.notion.site/Full-Virtualization-vs-Para-Virtualization-cd4933792f6a4a2b871a385f58592955)
