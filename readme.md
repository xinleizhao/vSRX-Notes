vSRX Notes
==========

1. [ Environment ](#environment)
1. [ Prepare your server ](#prepare)
1. [ Installation of vSRX ](#install)
1. [ Chassis Cluster ](#chassis)
1. [ Performance Tuning ](#performance)
1. [ Troubleshooting ](#faq)



<a name="environment"></a>
## Environment


```
Ubuntu 16.04.4 LTS

Install KVM
# sudo apt-get install qemu-kvm libvirt-bin virtinst bridge-utils cpu-checker

# virsh version
Compiled against library: libvirt 1.3.1
Using library: libvirt 1.3.1
Using API: QEMU 1.3.1
Running hypervisor: QEMU 2.5.0

vSRX 18.2R1

# lscpu
Architecture:          x86_64
Model name:            Intel(R) Xeon(R) CPU E5-2620 v4 @ 2.10GHz

# lspci | grep Eth
01:00.0 Ethernet controller: Intel Corporation Ethernet Controller X710 for 10GbE SFP+ (rev 01)
01:00.1 Ethernet controller: Intel Corporation Ethernet Controller X710 for 10GbE SFP+ (rev 01)
01:00.2 Ethernet controller: Intel Corporation Ethernet Controller X710 for 10GbE SFP+ (rev 01)
01:00.3 Ethernet controller: Intel Corporation Ethernet Controller X710 for 10GbE SFP+ (rev 01)
05:00.0 Ethernet controller: Intel Corporation I350 Gigabit Network Connection (rev 01)
05:00.1 Ethernet controller: Intel Corporation I350 Gigabit Network Connection (rev 01)
05:00.2 Ethernet controller: Intel Corporation I350 Gigabit Network Connection (rev 01)
05:00.3 Ethernet controller: Intel Corporation I350 Gigabit Network Connection (rev 01)

XL710 (with 40G port) work for PCI Passthrough, as stated in documentation. And we tried X710 (with 10G port) can do it too.
```
<a name="prepare"></a>
## Prepare your server

Please follow these steps in the documentation to prepare the server.

https://www.juniper.net/documentation/en_US/vsrx/topics/task/installation/security-vsrx-kvm-server-prep.html


<a name="install"></a>
## Installation of vSRX

```
# mkdir /home/hdd
# mount -t ext4 /dev/sdb1 /home/hdd
root@ubuntu:/home/hdd# virsh pool-define-as --name tmppool --type dir --target /home/hdd
Pool tmppool defined
 
root@ubuntu:/home/hdd# virsh pool-start tmppool
Pool tmppool started
 
root@ubuntu:/home/hdd# virt-install --name vSRXVM --ram 16384 --cpu SandyBridge,+vmx --vcpus=9 --arch=x86_64 --disk path=/home/hdd/junos-media-vsrx-x86-64-vmdisk-18.2R1.9.qcow2
,size=16,device=disk,bus=ide,format=qcow2 --os-type linux --os-variant rhel7 --import
 
Escape character is ^]
 
root@ubuntu:/home/hdd# virsh list --all
 Id    Name                           State
----------------------------------------------------
 3     vSRXVM                         running
 
 
root@ubuntu:/home/hdd# virsh console 3
Connected to domain vSRXVM
Escape character is ^]
 
 
Amnesiac (ttyd0)
 
login: root
 

root@% cli
root> show interfaces terse 
Interface               Admin Link Proto    Local                 Remote
gr-0/0/0                up    up
ip-0/0/0                up    up
lt-0/0/0                up    up
mt-0/0/0                up    down
sp-0/0/0                up    down
sp-0/0/0.0              up    down inet    
                                   inet6   
sp-0/0/0.16383          up    down inet    
dsc                     up    up
em0                     up    up
em0.0                   up    up   inet     128.0.0.1/2     
em1                     up    up
em1.32768               up    up   inet     192.168.1.2/24  
em2                     up    up
em2.0                   up    up  

ge- interfaces will appear after vSRX boots up for a while.

```


<a name="chassis"></a>
## Chassis Cluster

Chassis Cluster Topology is as follows:

![topology](https://git.juniper.net/xinleizhao/vSRX-Notes/raw/master/topo.png)
To use two hosts form a chassis cluster, two 1G eth links and two X710/XL710 ports will be used. 

1G links work as control and fabric, using VirtIO. Refer to <https://www.juniper.net/documentation/en_US/vsrx/topics/task/multi-task/security-vsrx--kvm-chassis-cluster-configuring.html>

```
# brctl addif virbr1 enp5s0f1
# brctl addif virbr2 enp5s0f2
# brctl show
bridge name	bridge id		STP enabled	interfaces
virbr0		8000.fe5400b2ae22	yes		vnet0
virbr1		8000.a0369febd431	yes		enp5s0f1
				                        vnet1
virbr2		8000.a0369febd432	yes		enp5s0f2
							            vnet2
# ifconfig vnet2 mtu 9000
# ifconfig enp5s0f2 mtu 9000
```

X710 port is for traffic, using PCI Passthrough. Refer to <https://www.juniper.net/documentation/en_US/vsrx/topics/task/configuration/security-vsrx-kvm-add-pci-passthrough.html>

Enable System for PCI Passthrough - Ubuntu (From <https://gbe0.com/networking/juniper/vmx/error-intel-iommu-disabled> )

1. Edit the file /etc/default/grub.
1. Look for the two GRUB_CMDLINE_LINUX lines that look like this by default:
   ```
   GRUB_CMDLINE_LINUX_DEFAULT=""
   GRUB_CMDLINE_LINUX=""
   ```
1. Add the intel_iommu=on option to the end of the GRUB_CMDLINE_LINUX_DEFAULT variable. In my case I also have huge pages enabled due to the amount of RAM for the vFP, so the modified lines look like this:
   ```
   GRUB_CMDLINE_LINUX_DEFAULT="processor.max_cstates=1 idle=poll pcie_aspm=off intel_iommu=on"
   GRUB_CMDLINE_LINUX="default_hugepagesz=1G hugepagesz=1G hugepages=64"
   ```
1. Save the updated configuration file.
1. Generate the new grub configuration file:
   ```
   update-grub
   ```
1. Reboot the host to apply the changes.


Also need to set number of queues to 8 **only** on HA fabric link. Refer to <https://www.juniper.net/documentation/en_US/vsrx/topics/task/multi-task/security-vsrx-with-kvm-scaling.html>

When chassis cluster is working, the VM XML file and vSRX configuration looks like below:

[Chassis cluster VM XML file](https://git.juniper.net/xinleizhao/vSRX-Notes/blob/master/vSRXVM.xml)

[Chassis cluster vSRX configuration](https://git.juniper.net/xinleizhao/vSRX-Notes/blob/master/chassis.conf)

Better to test chassis cluster without Z traffic. To achieve that, user need to configure only one redundancy group, i.e. active/passive.



<a name="performance"></a>
## Performance Tuning 

Credit to Karthik Kumar <ktkumar@juniper.net>

1. Better to use a server with more core’s per socket for max performance. Currently the server is R730, 8 cores per socket is not enough.

1. Disable Hyper-threading.
```
user@host# echo 0 > /sys/devices/system/cpu/cpu16/online
user@host# echo 0 > /sys/devices/system/cpu/cpu18/online
echo 0 > /sys/devices/system/cpu/cpu20/online
echo 0 > /sys/devices/system/cpu/cpu22/online
echo 0 > /sys/devices/system/cpu/cpu24/online
echo 0 > /sys/devices/system/cpu/cpu26/online
echo 0 > /sys/devices/system/cpu/cpu28/online
echo 0 > /sys/devices/system/cpu/cpu30/online
echo 0 > /sys/devices/system/cpu/cpu17/online
echo 0 > /sys/devices/system/cpu/cpu19/online
echo 0 > /sys/devices/system/cpu/cpu21/online
echo 0 > /sys/devices/system/cpu/cpu23/online
echo 0 > /sys/devices/system/cpu/cpu27/online
echo 0 > /sys/devices/system/cpu/cpu29/online
echo 0 > /sys/devices/system/cpu/cpu31/online 
```
1. Below settings in vm xml could improve the performance
```xml
  <memoryBacking>
    <hugepages/>
  </memoryBacking>
  <vcpu placement='static' cpuset='1,0-14'>9</vcpu>
  <cputune>
    <vcpupin vcpu='0' cpuset='1'/>
    <vcpupin vcpu='1' cpuset='0'/>
    <vcpupin vcpu='2' cpuset='2'/>
    <vcpupin vcpu='3' cpuset='4'/>
    <vcpupin vcpu='4' cpuset='6'/>
    <vcpupin vcpu='5' cpuset='8'/>
    <vcpupin vcpu='6' cpuset='10'/>
    <vcpupin vcpu='7' cpuset='12'/>
    <vcpupin vcpu='8' cpuset='14'/>
  </cputune>
  <numatune>
    <memory mode='strict' nodeset='0'/>
  </numatune>
```
1. Here are the parameters to pass to the Linux Kernel at boot in: /etc/default/grub:
 
```GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt intel_pstate=disable isolcpus=2-15default_hugepagesz=1G hugepagesz=1G hugepages=192 transparent_hugepage=never"```

Then # update-grub and reboot server.

After that run this bash script at boot time to ensure Intel Perf governer is off:
   
```bash
cpu=`cat /proc/cpuinfo  | grep MHz | wc | awk  '{print $1}'`
 
performance_mode ()
{
for i in `seq 0 $cpu`;
    do
        if [ -f /sys/devices/system/cpu/cpu$i/cpufreq/scaling_governor ]; then
            echo performance > /sys/devices/system/cpu/cpu$i/cpufreq/scaling_governor 2> /dev/null
        fi
        echo $i
  done
}
```

<a name="faq"></a>
## Troubleshooting


1. For Ubuntu 16.04, virt-manager will show "Wind River Linux 6.0.0.15 localhost console INIT: Sending processes the TERM signal".

    Solution: virt-manager is the Graphical console, please select the serial1 console from the dropdown menu which displays text console. Or use virsh console to see vSRX screen.

1. On chassis cluster, control link was down and two servers lose each other.

    This happens when PCI Passthrough is configured as chassis cluster control and fabric links. Chassis cluster control and fabric links need to be configured as VirtIO.

1. vSRX on chassis cluster are shown as VSRX-S when running "show chassis hardware".

    This happens when either the number of queues are not set to 8 on HA cluster, or fxp0 VirtIO interfaces number of queues are 8. fxp0 VirtIO interfaces number of queues should not be set. Also there’s no need to set number of queues for HA control link. Fab links is managed by RT flow thread and as a virtio interface the number of queues should be configured. Also, mtu should be set to 9000 for HA fab links.
    
1. On host server, error message "error: operation failed: Active console session exists for this domain" was seen.

    This can be solved by restarting libvirt-bin ```# /etc/init.d/libvirt-bin restart```
    
1. Testing with UDP 1500 frame size, there are some packet drops about usually 0.02% when tester gives 8000Mbps traffic.

    Please see Performance Tuning part of this document
    
1. Data transfer ports show "Down" after 10G copper cable disconnect and connecting back.

    Please use fiber cable instead of copper cable (10G) for reth failover testing. Interface will show "Up" and "Down" in real time if fiber cables are used.

1. How to check chassis cluster CPU performance
    ```
    root@vSRX-1> start shell user root 
    root@vSRX-1:~ # vty node0.fpc0
    

    TOR platform (2099 Mhz Intel(R) Xeon(R) processor, 7168MB memory, 16384KB flash)

    FLOWD_VSRX_L(vSRX-1 vty)# show i386 cpu
    CPU   Util WUtil Status SchedCounter                    
    1     2     0     alive  3699        
    2     2     0     alive  3699        
    3     2     0     alive  3699        
    4     2     0     alive  3699        
    5     2     0     alive  3699        
    6     2     0     alive  3699        
    7     2     0     alive  3699        
    8     3     0     alive  3699        
    Average [cpu0-7](  2) (  0)
    ```
1. On CentOS, PCI Passthrough cannot be enabled.

    Haven't found a solution yet.