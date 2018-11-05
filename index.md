## Memory-loaded python anywhere

A while back whiff of inspiration led me to research interesting problem. I knew of [pupy](https://github.com/n1nj4sec/pupy/) for some time and there were things i did not like. Most unfortunate (mis)feature of `pupy` is depending on python 2.7 forever.. It is year 2017 and i am one of those people who constantly advocate switching to python 3. So i thought to myself - i can do better. Here is a method how to load a modern python (3.6+) interpreter all from memory, supporting native extensions and running on anything from Windows XP and up. [Read More...](pages/memory-loaded-python-anywhere.md)

## Gaming VM performance

Thanks to advances of QEMU and virtio nowdays it is very easy to set up a VM with dedicated GPU. Great times for Linux gamers. However many face another issue - poor VM performance. I spent many times gathering bits on the internet and came up with a solution that works for me. [Read More...](pages/gaming-vm-performance.md)

## Python3 asyncio - call async code from synchronous code

Problem stems from the fact that while we can call `def` functions from `async def` functions, we can not do opposite. This essentially divides python code into two islands - async is land and synchronous one. Problems start to arise when we need to call async code from synchronous code which we do not control. [Read More...](pages/python3-asyncio-sync-async.md)

## Full Software KVM switch for VMs

I already wrote on [Sharing HID devices with KVM virtual machine](/#!pages/kvm-hid.md), but this is way better. It solves same problem but more elegantly and painless than ever. For description of the problem read original article. This article assumes you have to GPUs one of which is dedicated to host and another is dedicated to VM. You use one main monitor that has multiple outputs. Two outputs are connected one to each GPU. Monitor must be i2c-capable! [Read More...](pages/full-software-kvm-switch.md)

## Second X server with second GPU

Using GPU passthrough we have to assign a dedicated GPU to virtual machine. It is not unusual to give linux system the weak GPU (be it intel integrated graphics or just some old card) and dedicate the performant GPU to Windows VM. However what if we do need some GPU power on our main GNU/Linux OS? [Read More...](pages/second-xserver-second-gpu.md)

## Sharing HID devices with KVM virtual machine

I run KVM virtual machines for various things. Occasional gaming is one of those things. After i assembled new intel-based build (sorry AMD, buldozzer just did not cut it =\) i noticed a critical difference from my AMD build. On Sabertooth 990FX R2 back panel had different sets of USB sockets handled by different USB controllers. What i did was pass-through one USB controller to VM and it gave exclusive access of several USB ports to VM. Pair that with KVM switch and i had easy way to flip between VM and host. With Gigabyte GA-X99-UD4 motherboard all usb slots on back panel belong to same USB controller. Other two USB controllers handle internal USB headers and what not. Now this is a problem. I do not want to take away PC case front USB ports from host and i surely can not reserve entire back panel for VM. Libvirt to the rescue! [Read More...](pages/kvm-hid.md)
