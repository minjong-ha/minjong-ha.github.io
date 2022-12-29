---
layout: posts
title:  "Python 3 dbus-next: Implement dbus interface with asyncio"
author: Minjong Ha
published: true
date:   2022-11-29 09:00:00 +0900
---

In this post, I will explain how to implement and use dbus through python with asyncio (including unittest example).


## Introduction

DBUS is a message bus system, a simple way for applications to talk to one another[DBUS](https://www.freedesktop.org/wiki/Software/dbus/).
It provides IPC for user and system application (per-user-login-session daemon and system daemon).
Developer could register its own dbus interface and implement callback functions. 

Since I can check various C examples for dbus, I decide to write it for python.
There is a few references of dbus for python especially with asyncio.
I hope that this post can be help for fragmented dbus references in python.

I will not talk about asyncio itself in this post.
If you want to know what is the asyncio and how to use it, reference [it](https://docs.python.org/3/library/asyncio.html)


## Why dbus-next?

[pydbus](https://pydbus.readthedocs.io/en/latest/legacydocs/tutorial.html) is more familiar to the python users and has more references.
However, as far as I know, it does not support asyncio and less flexible to use.
[dbus-next](https://python-dbus-next.readthedocs.io/en/latest/index.html) supports asyncio and easy to add new methods or properties.

### Implement dbus interface through dbus-next

First, we need to implement a new dbus interface for other applications.
There are three compositions to deploy the interface: name, path (address), and interface

Name represents the unique connection name for bus daemon.
Usually, it has form of "org.example.dbustest".

Path (address) represents a connection to the bus daemon for application.
The application with dbus is a client and the dbus daemon is a server.
Usually, it has form of "/org/example/dbustest".
The address "/org/example/dbustest" specifies that the server will listen on a UNIX domain socket at the path "/org/example/dbustest".

Interface is an optional parameter for dbus.
It is useful if there are a lot of methods and properties for dbus, and they require being categorized.
Usually, it has form of "org.example.dbustest.Module".


Now, we need to implement dbus interface for bus daemon.
There are two files for register: xml and conf.
Since the name of dbus in this example is "org.example.dbustest", the name of two files are "org.example.dbustest.conf" and "org.example.dbustest.xml".
They should be located under the "/usr/share/dbus-1/system.d" directory.
The conf file represents the dbus information to the dbus daemon, and xml file describes the methods and properties that the dbus can handle.

```c
# org.example.dbustest.conf

<?xml version="1.0" encoding="UTF-8"?>

<!DOCTYPE busconfig PUBLIC
  "-//freedesktop//DTD D-BUS Bus Configuration 1.0//EN"
  "http://www.freedesktop.org/standards/dbus/1.0/busconfig.dtd">
<busconfig>
   <policy user="root">
     <allow own="org.example.dbustest"/>
   </policy>
   <policy context="default">
     <allow send_destination="org.example.dbustest"
            send_interface="org.example.dbustest.Module" />
     <allow send_destination="org.example.dbustest"
            send_interface="*" />
   </policy>
</busconfig>

```

Above content represents the .conf.
We can see that it is a system bus, and it has a destination with the interface.

```c
# org.example.dbustest.xml

 <!DOCTYPE node PUBLIC "-//freedesktop//DTD D-BUS Object Introspection 1.0//EN"
  "http://www.freedesktop.org/standards/dbus/1.0/introspect.dtd">
 <node name="/org/example/dbustest">
   <interface name="org.example.dbustest.Module">
     <method name="TestMethod">
       <arg type="s" direction="in"/>
       <arg type="a{sas}" direction="in"/>
       <arg type="b" direction="out"/>
     </method>
    <signal name="TestSignal">
       <arg type="a{ss}"/>
     </signal>
     <property name="TestProperty" type="a{ss}" access="readwrite"/>
   </interface>
</node>
```

Above content represents the .xml.
".xml" file is used in external application who wants to connect to dbus interface.
We can see there are three components: method, signal, and property.
External application can figure out the methods, signals, properties that dbus interface provides.


After register default dbus information for dbus daemon, it is time to implement an application who is connected to this dbus.
Following represents the requirements for python (dbus-next).


```python
import asyncio

from dbus_next.aio import MessageBus
from dbus_next.service import ServiceInterface, method, dbus_property, signal
from dbus_next import BusType

class TestModule(ServiceInterface):
    def __init__(self):
        super().__init__("org.example.dbustest.Module")

    ...

    @method()
    async def TestMethod(self, _arg: "s", device_list: "a{sas}") -> "s":
        for vm in self.vmlist.values():
            if vm.get_type() == _arg:
                for device in device_list.values():
                    xml = write_xml(device)
                    asyncio.create_task(vm.dom_device_detach(xml))
                return "Success"
        return "Fail : domain not exist"

    @signal()
    def TestSignal(self, _dict) -> "a{ss}":
        return _dict

    @dbus_property()
    def TestProperty(self) -> "a{ss}":
        return self.vmstate

    ...
```

"testModule" has inheritance to the "ServiceInterface", which is for dbus implementation.
And we can see there are @method, @signal, @dbus\_property that pre-defined in xml and conf files.

"@method" represents the function that the external application can call through the dbus.
I will explain about it in later section.

"@dbus\_property()" represents the property than the external application can call through the dbus.
Also, I will explain about it in later section.

"@signal()" is the only feature that should be called inside the dbus publishing process.
If the process calls @signal(), the signal is emitted and spread to the every applications connected with dbus.
It is possible to assign callback function for signal and I will explain it in later section.

One thing important is TestSignal has difference argument information with xml.
In xml, TestSignal only has one argument: dictionary for return while methods have same arguments.
It is because that the xml file is for the external application.
TestSignal() function itself should be called inside the application and external application only listen about it.
Thus, xml for external application only knows about the return value itself.
You don't need to specify the type of argument for signal function such as "s", "b", "n" and etc.


Now, what we have to do is publishing the dbus interface inside the application


```python
# Someplace in main...

    dbus_manager = TestModule()
    
    try:
        bus = MessageBus(bus_type=BusType.SYSTEM)
        await bus.connect()
        bus.export("/org/example/dbustest", dbus_manager)
        await bus.request_name("org.example.dbustest")
        await bus.wait_for_disconnect()
    except:
        sys.exit(1)
```

MessageBus() connects "dbus\_manager" object with system message bus.
"dbus\_manager" object will be exported with the interface name "/org/example/dbustest"
If another application requests method or property using designated dbus, "dbus\_manager" will answer.


### Interact with dbus-interface at the external application.

In external application, you should acquire the connection to the dbus first.


```python
async def _get_dbus_interface():
    bus = await MessageBus(bus_type=BusType.SYSTEM).connect()

    with open("../../dbus/org.example.dbustest.xml", "r") as f:
        introspection = f.read()

    proxy_object = bus.get_proxy_object(
        "org.example.dbustest", "/org/example/dbustest", introspection
    )

    interface = proxy_object.get_interface("org.example.dbustest.Module")

    return bus, interface

async def _on_test_signal():
    logging.info("I got signal from dbustest!")

async def test_dbus_backup_method(self):
    _bus, _if = await _get_dbus_interface()

    ret = await _if.call_test_method()
    _if.on_test_signal(_on_test_signal)

    # For waiting backup result
    await asyncio.sleep(10)
```


Above codes represents the requiring of dbus connection with custom interface in python (dbus-next).
Since we define the dbus information in .xml file, we can use it to connect with test dbus interface.
You can see there is a function of "\_if.call\_test\_method()" and "\_if.on\_test\_signal(self.\_on\_test\_signal)".
And you can also realize that the method I defined is TestMethod(), and the signal I defined is TestSignal().
"dbus-next" automatically convert the upper case to lower case with "\_".
And for method, it assigns "call" as an prefix.
Thus, If you call "call\_test\_method()" with argument it defined, the TestMethod() function will be called.

You can also see there is a function "\_if.on\_test\_signal()".
"on" is a prefix for dbus signal for assigning the callback function.
I defined a simple "\_on\_test\_signal()" function and pass it as an argument with "on\_test\_signal()".
If the process emits TestSignal(), application will listen and execute callback funtion it assigned.


<!--- Add property --->


## Appendix

<!-- unittest for asyncio -->


```python
from unittest import IsolatedAsyncioTestCase, main

async def _get_dbus_interface():
    bus = await MessageBus(bus_type=BusType.SYSTEM).connect()

    with open("../../dbus/org.example.dbustest.xml", "r") as f:
        introspection = f.read()

    proxy_object = bus.get_proxy_object(
        "org.example.dbustest", "/org/example/dbustest", introspection
    )

    interface = proxy_object.get_interface("org.example.dbustest.Module")

    return bus, interface

class AsyncTestCase(IsolatedAsyncioTestCase):
    def setUpClass():
        # DO SOMETHING

    def tearDownClass():
        # DO SOMETHING

    async def test_asyncio(self):
        await async_func_for_unittest()
    ...

if __name__ == "__main__":
    main()
```

Since the dbus-next supports asyncio dbus interface, unittest requires to support asyncio.
"unittest" supports asyncio with "IsolatedAsyncioTestCase".
There is no change between normal unittest and asyncio unittest, but add await for async functions.
