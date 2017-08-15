# Using same GPU for passthrough and X Server


## The problem

Using GPU passthrough we have to assign a dedicated GPU to virtual machine. It is not unusual to give linux system the weak GPU (be it intel integrated graphics or just some old card) and dedicate the performant GPU to Windows VM. However what if we do need some GPU power on our main GNU/Linux OS?


## The solution

While we can not use one GPU for both things at the same time - we can use it either on GNU/Linux or on Windows VM. All this without reboot. Trick is to start second X Server on our second more powerful GPU. This involves unbinding passthrough GPU from `vfio-pci` driver and binding it to vendor driver (for example `nvidia` in case of proprietary Nvidia driver). We also have to use custom X Server configuration to instruct X Server to use second GPU.

## Method

1. Find PCI address of your passthrough GPU by running `lspci` and finding proper entry.
```
    05:00.0 VGA compatible controller: NVIDIA Corporation GP104 [GeForce GTX 1080] (rev a1)
    05:00.1 Audio device: NVIDIA Corporation GP104 High Definition Audio Controller (rev a1)
```
2. Create custm X Server configuration in `/etc/X11/`. I created `/etc/X11/1080.conf` with contents:
```
    Section "Device"
        Identifier     "nvidia"
        Driver         "nvidia"       # Driver name we are going to use
        BusID          "PCI:5:0:0"    # Bus ID from lspci command
    EndSection
```
3. Create script `flipdriver`. This script unbinds pci device from current driver and binds it to specified driver. I lifted from http://pastebin.com/iPHL1KjV because efficiency.
```bash
    #!/usr/bin/bash

    if [ "$EUID" != "0" ];
    then
        echo "Please run as root."
        exit
    fi

    dev="$1"
    driver="$2"

    if [ -z $driver ] | [ -z $dev ];
    then
        return 1
    fi

    vendor=$(cat /sys/bus/pci/devices/${dev}/vendor)
    device=$(cat /sys/bus/pci/devices/${dev}/device)

    echo -n Unbinding $vendor:$device ...

    if [ -e /sys/bus/pci/devices/${dev}/driver ]; then
        echo ${dev} > /sys/bus/pci/devices/${dev}/driver/unbind
        while [ -e /sys/bus/pci/devices/${dev}/driver ]; do
            sleep 0.5
            echo -n .
        done
    fi
    echo " OK!"

    echo -n Binding \'$driver\' to $vendor:$device ...
    echo ${vendor} ${device} > /sys/bus/pci/drivers/${driver}/new_id

    echo " OK!"

    exit 0
```
4. Create script `start.sh` that will do actual work of driver rebinding and starting X Server:
```bash
    #!/usr/bin/bash

    if [ "$EUID" == "0" ];
    then
        echo "Please do not run as root."
        exit
    fi

    # Xorg shouldn't run.
    if [ -n "$( ps -C xinit | grep xinit )" ];
    then
        echo Don\'t run this inside Xorg!
        exit 1
    fi

    # Bind our GPU to vendor driver
    sudo ./flipdriver 0000:05:00.0 nvidia
    sudo ./flipdriver 0000:05:00.1 nvidia

    # Start X Server
    xinit /usr/bin/startkde -- :2 -xf86config 1080.conf vt2

    # After X Server exits bind our GPU back to vfio-pci driver so we can use it windows VM again
    sudo ./flipdriver 0000:05:00.0 vfio-pci
    sudo ./flipdriver 0000:05:00.1 vfio-pci
```
5. Press Ctrl+Alt+F2

6. Login to second user (not the user that runs your primary X Server!)

7. Run `./start.sh`

8. Switch your monitor to secondary GPU output.

And thats it. Now you should see new desktop session that runs on your secondary more powerful GPU. Now it is time to do some 3d rendering or maybe linux gaming.

Be aware:

* X Server configuration files must be in `/etc/X11/` directory.
* `xinit /usr/bin/startkde -- :2 -xf86config 1080.conf vt2` is specific to my system. You may use binary of your demanding application in place of `/usr/bin/startkde`, or other virtual terminals, but you should change `vt2` accordingly (or maybe use `$XDG_VTNR` environment variable).
* Input follows virtual terminal. In order to control your second desktop session you must be switched to virtual terminal that started this session.

[gimmick:Disqus](tech-notebook)
