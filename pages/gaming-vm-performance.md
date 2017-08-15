Performance of your gaming VM
=============================

Thanks to advances of QEMU and virtio nowdays it is very easy to set up a VM with dedicated GPU. Great times for Linux gamers. However many face another issue - poor VM performance. I spent many times gathering bits on the internet and came up with a solution that works for me.

## Hardware and Software

* [i7-6800K](http://ark.intel.com/products/94189/Intel-Core-i7-6800K-Processor-15M-Cache-up-to-3_60-GHz) 6 core 12 thread CPU
* [GA-X99-UD4](https://www.gigabyte.com/Motherboard/GA-X99-UD4-rev-11) motherboard
* 32 GB or RAM
* GTX 1080
* Archlinux
* [cpuset](https://aur.archlinux.org/packages/cpuset/)
* [ovmf](https://www.archlinux.org/packages/extra/any/ovmf/)
* [libvirt-python2](https://www.archlinux.org/packages/community/x86_64/libvirt-python2/)
* Python 2.7

## The Setup

My goal was to allocate slice of my hardware to VM when it boots, and release these resources on VM shutdown so that host machine can run at full speed when needed. For that purpose i use libvirt hook script that does just that on machine boot. It also allcates required amount of hugepages and releases them when last VM needing hugepates shuts down. VM gets 4 cores, 8 threads and host has 2 cores, 4 threads. All tasks are migrated from VM cores to host cores so they do not interfere with applications VM is executing.

## Configuration

First we have to set some variables in qemu hook script:

```sh
    TOTAL_CORES='0-11'
    TOTAL_CORES_MASK=FFF            # 0-11, bitmask 0b111111111111
    HOST_CORES='0-1,6-7'            # Cores reserved for host
    HOST_CORES_MASK=C3              # 0-1,6-7, bitmask 0b000011000011
    VIRT_CORES='2-5,8-11'           # Cores reserved for virtual machine(s)
```

QEMU enumerates cores from 0. Since i run them hyperthreaded. As you see from config `HOST_CORES` is set to have four threads that run on two cores. Threads 0 and 6 run on the same core, threads 1 and 7 run on another core. This is very important for performance. If you were to assign two virtual cores sharing physical core one to guest and one to VM then performance would suffer. To get a sense what cores are siblings just do `cat /proc/cpuinfo|grep "core id"`. Matching core id means thread runs in this same physical core.

Then make sure key settings are set in your libvirt VM. Enable hugepages and pin your VM's cores. As you can see i have four hyperthreaded cores assigned to VM, total 8 threads. Key trick here is to have sibling cores follow each other in configuration. QEMU assumes that sibling cores follow one another: core 0 and core 1 are siblings, therefore they must be pinned to sibling cores on host, which are core 2 and core 8. I also pin emulator threads to cores assigned to host so that emulator does not stall VM.

Make sure you have `hugetlbfs /dev/hugepages hugetlbfs defaults` line in `/etc/fstab`. This enables hugepages.

Other features and clock settings enable goodies VM depends on to speed up execution. This includes dummy `vendor_id` for working around NVIDIA driver not running in a VM.

```xml
  <memoryBacking>
    <hugepages/>
  </memoryBacking>
  <vcpu placement='static'>8</vcpu>
  <cputune>
    <vcpupin vcpu='0' cpuset='2'/>
    <vcpupin vcpu='1' cpuset='8'/>
    <vcpupin vcpu='2' cpuset='3'/>
    <vcpupin vcpu='3' cpuset='9'/>
    <vcpupin vcpu='4' cpuset='4'/>
    <vcpupin vcpu='5' cpuset='10'/>
    <vcpupin vcpu='6' cpuset='5'/>
    <vcpupin vcpu='7' cpuset='11'/>
    <emulatorpin cpuset='0-1,6-7'/>
  </cputune>
  <features>
    <acpi/>
    <apic/>
    <hyperv>
      <relaxed state='on'/>
      <vapic state='on'/>
      <spinlocks state='on' retries='8191'/>
      <vendor_id state='on' value='whatever'/>
    </hyperv>
    <kvm>
      <hidden state='on'/>
    </kvm>
  </features>
  <cpu mode='host-passthrough' check='none'>
    <topology sockets='1' cores='4' threads='2'/>
  </cpu>
  <clock offset='utc'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
    <timer name='kvmclock' present='yes'/>
    <timer name='hypervclock' present='yes'/>
  </clock>
```

Also i should note that all of this is achieved using UEFI boot through ovmf firmware. To enable it for your VM just set `loader` and `nvram`:

```xml
  <os>
    <type arch='x86_64' machine='pc-i440fx-2.9'>hvm</type>
    <loader readonly='yes' type='pflash'>/usr/share/ovmf/x64/ovmf_code_x64.bin</loader>
    <nvram template='/usr/share/ovmf/x64/ovmf_code_x64.bin'>/var/lib/libvirt/qemu/nvram/win-vars.bin</nvram>
  </os>
```

## Full scripts

Script that orchestrates resource allocation and release. Change variables at start of the script as needed.

`/etc/libvirt/hooks/qemu`:
```sh
#!/usr/bin/env bash
# This script dynamically manages allocated hugepages size depending on running libvirt VMs.
# Based on Thomas Lindroth's shell script which sets up host for VM: http://sprunge.us/JUfS
# put this script to /etc/libvirt/hooks/qemu

TOTAL_CORES='0-11'
TOTAL_CORES_MASK=FFF            # 0-11, bitmask 0b111111111111
HOST_CORES='0-1,6-7'            # Cores reserved for host
HOST_CORES_MASK=C3              # 0-1,6-7, bitmask 0b000011000011
VIRT_CORES='2-5,8-11'           # Cores reserved for virtual machine(s)

HUGEPAGES_SIZE=$(grep Hugepagesize /proc/meminfo | awk {'print $2'})
HUGEPAGES_SIZE=$((HUGEPAGES_SIZE * 1024))
HUGEPAGES_ALLOCATED=$(sysctl vm.nr_hugepages | awk {'print $3'})

VM_NAME=$1
VM_ACTION=$2

shield_vm() {
    cset set -c $TOTAL_CORES -s machine.slice
    # Shield two cores cores for host and rest for VM(s)
    cset shield --kthread on --cpu $VIRT_CORES
}

unshield_vm() {
    echo $TOTAL_CORES_MASK > /sys/bus/workqueue/devices/writeback/cpumask
    cset shield --reset
}

# For manual invocation
if [[ $VM_NAME == 'shield' ]];
then
    shield_vm
    exit 0
elif [[ $VM_NAME == 'unshield' ]];
then
    unshield_vm
    exit 0
fi

cd $(dirname "$0")
VM_HUGEPAGES_NEED=$(( $(./vm-mem-requirements $VM_NAME) / HUGEPAGES_SIZE ))

if [[ $VM_ACTION == 'prepare' ]];
then
    echo 3 > /proc/sys/vm/drop_caches
    echo 1 > /proc/sys/vm/compact_memory

    VM_HUGEPAGES_TOTAL=$(($HUGEPAGES_ALLOCATED + $VM_HUGEPAGES_NEED))
    sysctl vm.nr_hugepages=$VM_HUGEPAGES_TOTAL

    if [[ $HUGEPAGES_ALLOCATED == '0' ]];
    then
        shield_vm
        # Reduce VM jitter: https://www.kernel.org/doc/Documentation/kernel-per-CPU-kthreads.txt
        sysctl vm.stat_interval=120

        sysctl -w kernel.watchdog=0
        # the kernel's dirty page writeback mechanism uses kthread workers. They introduce
        # massive arbitrary latencies when doing disk writes on the host and aren't
        # migrated by cset. Restrict the workqueue to use only cpu 0.
        echo $HOST_CORES_MASK > /sys/bus/workqueue/devices/writeback/cpumask
        # THP can allegedly result in jitter. Better keep it off.
        echo never > /sys/kernel/mm/transparent_hugepage/enabled
        # Force P-states to P0
        echo performance | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
        echo 0 > /sys/bus/workqueue/devices/writeback/numa
        >&2 echo "VMs Shielded"
    fi
fi

if [[ $VM_ACTION == 'release' ]];
then
    VM_HUGEPAGES_TOTAL=$(($HUGEPAGES_ALLOCATED - $VM_HUGEPAGES_NEED))
    VM_HUGEPAGES_TOTAL=$(($VM_HUGEPAGES_TOTAL<0?0:$VM_HUGEPAGES_TOTAL))
    sysctl vm.nr_hugepages=$VM_HUGEPAGES_TOTAL

    if [[ $VM_HUGEPAGES_TOTAL == '0' ]];
    then
        # All VMs offline
        sysctl vm.stat_interval=1
        sysctl -w kernel.watchdog=1
        unshield_vm
        echo always > /sys/kernel/mm/transparent_hugepage/enabled
        echo powersave | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
        echo 1 > /sys/bus/workqueue/devices/writeback/numa
        >&2 echo "VMs UnShielded"
    fi
fi
```

Python script for estimating VM memory requirements, used for dynamic allocation of hugepages. I agree that it would be better to have this functionality in main shellscript, however implementing xml parsing in bash is beyond my abilities.

`/etc/libvirt/hooks/vm-mem-requirements`:
```py
#!/usr/bin/python2
import os
import sys
import argparse
from xml.etree import ElementTree

config_path = '/etc/libvirt/qemu'
units = {
    'b': 1,
    'bytes': 1,
    'KB': 10**3,
    'k': 2**10,
    'KiB': 2**10,
    'MB': 10**6,
    'M': 2**20,
    'MiB': 2**20,
    'GB': 10**9,
    'G': 2**30,
    'GiB': 2**30,
    'TB': 10**12,
    'T': 2**40,
    'TiB': 2**40
}

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('vm_name')
    args = parser.parse_args()
    with open(os.path.join(config_path, args.vm_name + '.xml')) as fp:
        vm_xml = ElementTree.fromstringlist(fp.read())
        if vm_xml.find('./memoryBacking/hugepages') is not None:
            mem = vm_xml.find('./memory')
            try:
                multipier = units[mem.attrib['unit']]
            except KeyError:
                multipier = 1
            print(int(mem.text) * multipier)
            sys.exit(0)
    sys.exit(-1)
```

## The Result

Result is buttery-smooth Battlefield 1 experience reaching 144 FPS.

[gimmick:Disqus](tech-notebook)
