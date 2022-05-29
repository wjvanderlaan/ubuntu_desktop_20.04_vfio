# ubuntu_desktop_20.04_vfio

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

