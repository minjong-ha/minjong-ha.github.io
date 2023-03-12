---
layout: posts
title:  "Host-Guest Communication: Full vs Para Virtualization - 2"
author: Minjong Ha
published: true
date:   2022-04-25 19:13:43 +0900
---

## Introduction

In previous post, we analyzed how the vCPU thread working and understoodd the concept of the vCPU in the GUEST_MODE under the QEMU-KVM hypervisor.
Now, lets look around how Para-Virtualization (virtio) works and compare it to the Full-Virtualization.
Since we already understand the detail operations, comparison between the Full and Para Virtualization is much easier.

## Full-Virtualization

<!-- require proper image-->
<img data-action="zoom" src='{{ "../assets/images/posts/2022-04-25-host-guest-communication/QEMU-KVM.png" | relative_url }}' alt='relative'>

Assume that we try to send network packet inside the Guest.
In the Guest, the application requests to the Guest's Kernel and hands over the data to make it as a packet and send.
However, in Full-Virtualization, Guest does not know that the network device is a emulated, virtual device.
The GUEST's kernel will behave as if the device were a real physical device, which means it will write the data to the MMIO area of the device.
Since the device is not a real, The host's help is required to transfer the guest's data to the actual, physical device.
It requires work on a host that can access the physical device.

<!-- It is not sure-->
To handle it, QEMU-KVM implements a BIOS to trigger the __vmexit__ when the write is made to the memory area allocated to the virtual device.
As I already mentioned in the previous post, the KVM returned from the GUEST_MODE through the vmexit tries to figure out the reason of it.
Soon, it discovers that a device working for the Guest has a data to handle.
KVM __copies__ the data inside the device's memory area to the kernel space (1st user-kernel data copy).
However, since the device itself is a thread in the user area, KVM (kernel) could not handle it over to the device.
KVM should return the result of the ioctl() that the QEMU had called and let the QEMU handles it.
Now, QEMU looks up the reason of vmexit.
It figures out the vmexit is triggered since the device request, and __copies__ the data from the kernel area (2nd user-kernel data copy).
Finally, QEMU __passes over (copies)__ the data to the native kernel drive and it causes 3rd user-kernel data copy.

If we look at the result, data about the device in the guest area is only moved to the host, but actually, it can be seen that unnecessary data copy (overhead) happens because the guest does not know that it is on the virtualization.

## Para-Vitualization

<!-- require proper image-->

<img data-action="zoom" src='{{ "../assets/images/posts/2022-04-25-host-guest-communication/VIRTIO.png" | relative_url }}' alt='relative'>

Not only the vmexits, user-kernel data copy is the huge overheads.
It is because that the Guest does not know the device is a virtual, emulated device.
Para-Virtualization (Virtio) presents virtualization-aware devices and dedicated drivers based on the virtio based communication.

Virtio has a shared memory area between the Host and Guest since the device awares it is on the virtualization.
Frontend virtio device driver in the Guest, and the backend virtio device driver in the Host share the memory area and implement data structure called virtqueues (v-rings).
They communicate each other through the virtqueues.
Unlike the Full-Virtualization copies data from user to kernel and kernel to user, virtio only kicks a notification.
The data itself is located in the virtqueue.
The notification notices there is a data that Host or Guest should read, and it takes the data from the virtqueue.

Still, virtio causes vmexit to context switch to the Host from the Guest.
However, it could reduces the copy overheads compared to the Full-Virtualization.

<!---
## Experiment
--->

## Conclusion

Virtio reduces the overhead from the user-kernel data copying through the virtqueue (v-ring) based communication.
Virtio presents various drivers for developer's convinience and efficiency using virtqueue, such as virtio-blk, virtio-serial, virtio-pci.
For instancde, QEMU-Guest-Agent (qga) is implemented based on the virtio-serial.

There are more interesting technologies and implementations based on vhost, vhost-user which are the more recent Para-Virtualization techniques.
For example, vSock (Virtual Socket) connects the Host and Guest's sockets and supports the application communication.
I hope there will be a chance to analyze about it.
