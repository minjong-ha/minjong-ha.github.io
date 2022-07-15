---
layout: posts
title:  "virtio-serial Linux Host - Windows Guest"
author: Minjong Ha
published: false
date:   2022-06-03 16:13:43 +0900
---

# Virtio-Serial Appeareance in Linux Host and Windows Guest
<!-- What is the virtio-serial?-->
Virtio presents the communication channels between the Host and Guest.
Host and Guest share the memory space and send notification to each others to notify that there are the data that the Host or Guest should read.
For example, if the Host tries to send some data to the Guest, it writes the data it want to send in the virtqueue (v-ring) located in the shared memory and gives a notification to the Guest through the KVM.
The Guest's virtio driver now knows that there are data it should read.
In this operations, virtio provides the communication API using the virtqueue.
Since the Host and Guest both require communication handler, they should install the virtio-aware driver based on the virtio APIs.

Implementing drivers for both Host and Guest is the time consuming works.
Fortunately, Virtio also presents the ready-to-use drivers for the various purposes: virtio-blk, virtio-pci, virtio-serial, and etcs.
In this post, I will explain about the virtio-serial drivers.

# Preparation

```xml
<channel type='unix'>
  <source mode='bind' path='/var/lib/libvirt/qemu/f16x86_64.agent'/>
  <target type='virtio' name='org.qemu.guest_agent.0'/>
</channel>
```

There are two steps for using virtio-serial: Install virtio-serial driver, and specify virtio-serial channel in the libvirt domain xml.
Unlike the virtio-serial driver is already installed in the Linux as a kernel configuration, Windows does not support virtio frontend drivers in the initial state.
The user should install the virtio drivers in manually at [here](https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md)
Also, the user should notifies that the vm has virtio devices to the hypervisor.
Since I manage the VMs in my machine through libvirt, I explain how to add the virtio device on the hypervisor based on it.
Above xml codes represents the example of qemu-guest-agent specification, which is the one of the virtio-serial based guest agent.
(oVirt guest agent also uses virtio-serial channel for communication)

In the xml codes, we can see the two components: source and target.
'source' represents the socket information on the Linux Host.
Application in the Host can connect to the socket in the 'source' and works as a client.
'target' represents the name of port in the Windows Guest.
I will explain details about it through the last sections.


# Linux Host socket connection with QEMU
In the Linux Host, QEMU presents a socket for a channel to communicate with the Guest.
QEMU hypervisor performs a role as a server, and a process which tries to connect to the socket is a client.
Since the QEMU only accept one client, it is impossible connecting multiple processes to the socket unless modifies the source codes of QEMU (It is not sure because I did not try it).
You can use the socket through the Linux socket API (open, connect, listen , receive, send, and etcs).


# Windows Guest port connection with WIN32 API
In the Windows Guest, QEMU presents a port for a channel to communicate with the Guest.
In fact, QEMU also provides a port for a channel in the Linux Guest either.
However, in this post, I only explain about the Windows Guest since there are many references for the Linux Guest.

You can use the virtio-serial port in Windows Guest through the WIN32 API such as CreateFile, ReadFile, WriteFile and etcs.
However, there are unique characteristics (or restrictions) for virtio-serial port.
In my experience, it is impossible to use SetCommMask(), WaitForEvent().
It means that I can't implement event-driven port communication with virtio-serial and configurate timeout.
I do not know why it is not enable (there is no official document or reference for it).
However, since there are some mentions about these problems in different situation, I assume that the driver does not support full WIN32 API.

<!-- with characteristics compare with orninary port in WIN32 API -->


# Example
Below codes and images represent a simple example using virtio-serial port in the Windows Guest.
Unlike the qemu-guest-agent or oVirt-guest-agent work as host-driven services, I implemented it as a guest-driven agent, which means the Guest requests or sends commands to the Host.

<!-- add example codes and explanation-->
## Open the Socket and Serial-Port
<!-- host -->
```Python 3
 def open_all_sock(self):
        g2h_path = Xml().get_sock_path(self.__dom)

        if self.g2h_sock == None:
            self.g2h_sock = self.__open_sock(g2h_path)

```
In the host, as I mentioned in previous section, the 3rd party process can connect to the bind socket as a client.
The path of the socket is defined in the libvirt xml (at least in my working environment).
You can use ordinary socket API: open, close, read, write.
However, it is impossible to connect multiple clients at the same time.
In my analysis, QEMU only presents a single connection bind.

<!-- guest -->
```Python 3
 def __open_port(self, path):
        try:
            port = win32file.CreateFile(path, win32con.GENERIC_READ | win32con.GENERIC_WRITE,
                                        0,#win32con.FILE_SHARE_READ | win32con.FILE_SHARE_WRITE,
                                        win32security.SECURITY_ATTRIBUTES(),
                                        win32con.OPEN_EXISTING,
                                        win32con.FILE_FLAG_OVERLAPPED,
                                        0)
            return port

        except:
            error = windll.kernel32.GetLastError()
            _logger.debug("fail to open port. GetLastError: %d" % error)
            return None
```

In the guest, the communication channel is presented as a form of serial-port.
Unlike the normal serial-port is in the serial-port section as a COM in the device manager, virtio-serial port is in the hidden device - other device section.
In the above xml, Preparation section, I explained that the name in the target represents the name of the port inside the guest.
However, that name never appears inside the guest on the UI.
It only appears as a vport0n format no matter how many virtio-serial channel exists.

## send/recv and write/read
<!-- host -->
``` Python 3
  def __send_to(self, sock, msg):
        sock.send(msg.encode('utf-8'))

    def __recv_from(self, sock):
        recv_data = sock.recv(BUF_LEN).decode('utf-8')
        return recv_data
```

<!-- guest -->
``` Python 3
def __send_to(self, port, ovrlpd, msg):
        ret, nr_written = win32file.WriteFile(port, msg, ovrlpd)
        if ret == 997: #ERROR_IO_PENDING
            _logger.debug("I/O Pending ERROR in __send_to() is normal return. Continue")
        else:
            _logger.debug("Fail to WriteFile() with Error: %d" % ret)
            _logger.debug("WriteFile() can return 0 and ERROR_IO_PENDING")
            return False

        win32event.WaitForSingleObject(ovrlpd.hEvent, TIMEOUT) #(hHANDLE, milliseconds)

        try:
            win32file.GetOverlappedResult(port, ovrlpd, False)
        except:
            return False

        return True


    def __recv_from(self, port, ovrlpd):
        ret, buf = win32file.ReadFile(port, win32file.AllocateReadBuffer(BUF_LEN), ovrlpd)
        if ret == 997: #ERROR_IO_PENDING
            _logger.debug("I/O Pending ERROR in __recv_from() is normal return. continue")
        else:
            _logger.debug("Fail to ReadFile() Error: %d" % ret)
            _logger.debug("ReadFile() can return 0, ERROR_MORE_DATA or ERROR_IO_PENDING")

        win32event.WaitForSingleObject(ovrlpd.hEvent, TIMEOUT) #(hHANDLE, milliseconds)
        try:
            nr = win32file.GetOverlappedResult(port, ovrlpd, False)
        except:
            _logger.debug("Timeout for the __recv_from(). Timeout: %d msec" % TIMEOUT)
            return False

        buf = buf.tobytes()
        buf = buf[:nr]
        buf = buf.decode('utf-8')

        return buf

```

