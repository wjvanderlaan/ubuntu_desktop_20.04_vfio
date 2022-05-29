# ubuntu_desktop_20.04_vfio

    #################################################################################################  
    #  Scope :                                                                                      #  
    #  Ubuntu 20.04 Desptop and an NVIDIA GTX 960 to be used in a QEMU/KVM Windows virtual machine  #  
    #  Goal is to use the graphics card capabilities in Autodesk Fusion 360                         #  
    #################################################################################################

First add parameters to /etc/default/grub:

  AMD based :
    "amd_iommu=on iommu=pt kvm.ignore_msrs=1"  
    
  Intel based :
    "intel_iommu=on iommu=pt kvm.ignore_msrs=1"  

     sudo update_grub

Then reboot the system.

     reboot

Isolate nvidia graphics card (GTX 960), find the pci bus addresses:

     lspci -nn | grep -i nvidia  
 
Output:  
 
>  01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM206 [GeForce GTX 960] [10de:1401] (rev a1)  
>  01:00.1 Audio device [0403]: NVIDIA Corporation GM206 High Definition Audio Controller [10de:0fba] (rev a1)  

Create script "vfio-pci-override.sh".  
I placed the script in folder "/etc/initramfs-tools/conf.d".  
Add the pci addresses to the DEVS array:  

    #!/bin/sh
    
    DEVS="0000:01:00.0 0000:01:00.1"
    # "0000:09:00.0 0000:09:00.1"
    
    if [ ! -z "$(ls -A /sys/class/iommu)" ]; then
        for DEV in $DEVS; do
            echo "vfio-pci" > /sys/bus/pci/devices/$DEV/driver_override
            echo ${DEV} > /sys/bus/pci/drivers_probe
        done
    fi
    
    modprobe -i vfio-pci  # does'nt do anything, could be removed

Make the script executable :  

    sudo +x /etc/initramfs-tools/conf.d/vfio-pci-override.sh  

Create file "/etc/modprobe.d/vfio.conf" :  

    install vfio-pci /etc/initramfs-tools/conf.d/vfio-pci-override.sh  

Regenerate initramfs :  

    sudo update-initramfs -u  

Reboot the system and check if the driver used by both is "Kernel driver in use: vfio-pci".  

    sudo lspci -nnv

Output :  

    01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GM206 [GeForce GTX 960] [10de:1401] (rev a1) (prog-if 00 [VGA controller])
        Subsystem: ASUSTeK Computer Inc. GM206 [GeForce GTX 960] [1043:8520]
        Flags: fast devsel, IRQ 11, NUMA node 0
        Memory at f8000000 (32-bit, non-prefetchable) [disabled] [size=16M]
        Memory at b0000000 (64-bit, prefetchable) [disabled] [size=256M]
        Memory at ce000000 (64-bit, prefetchable) [disabled] [size=32M]
        I/O ports at ef00 [disabled] [size=128]
        Expansion ROM at f9000000 [virtual] [disabled] [size=512K]
        Capabilities: [60] Power Management version 3
        Capabilities: [68] MSI: Enable- Count=1/1 Maskable- 64bit+
        Capabilities: [78] Express Legacy Endpoint, MSI 00
        Capabilities: [100] Virtual Channel
        Capabilities: [250] Latency Tolerance Reporting
        Capabilities: [258] L1 PM Substates
        Capabilities: [128] Power Budgeting <?>
        Capabilities: [420] Advanced Error Reporting
        Capabilities: [600] Vendor Specific Information: ID=0001 Rev=1 Len=024 <?>
        Capabilities: [900] Secondary PCI Express
        Kernel driver in use: vfio-pci
        Kernel modules: nvidiafb, nouveau

    01:00.1 Audio device [0403]: NVIDIA Corporation GM206 High Definition Audio Controller [10de:0fba] (rev a1)
        Subsystem: ASUSTeK Computer Inc. GM206 High Definition Audio Controller [1043:8520]
        Flags: bus master, fast devsel, latency 0, IRQ 10, NUMA node 0
        Memory at f9ffc000 (32-bit, non-prefetchable) [size=16K]
        Capabilities: [60] Power Management version 3
        Capabilities: [68] MSI: Enable- Count=1/1 Maskable- 64bit+
        Capabilities: [78] Express Endpoint, MSI 00
        Kernel driver in use: vfio-pci
        Kernel modules: snd_hda_intel

In virtual machine manager (sudo virt-manager) check if xml editing is enabled.  
Add the pci devices to the virtual machine and edit the xml part to make sure the two devices are combined :  

Device PCI 0000:01:00.0 add multifunction='on':  

    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
      </source>
      <address type='pci' domain='0x0000' bus='0x09' slot='0x00' function='0x0' multifunction='on'/>
    </hostdev>
    
Device PCI 0000:01:00.1, make bus number the same as PCI 0000:01:00.0, change function from 0x0 to 0x1 :  
    
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x01' slot='0x00' function='0x1'/>
      </source>
      <address type='pci' domain='0x0000' bus='0x09' slot='0x00' function='0x1'/>
    </hostdev>
 






