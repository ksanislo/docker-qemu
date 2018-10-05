# Docker container ksanislo/qemu
A performance oriented docker container for QEMU/KVM

Based on https://github.com/tianon/dockerfiles/tree/master/qemu

There are substantial descriptive comments about configuration in environment variables at the top of the start-qemu script. Look there for up-to-date info on everything that can be tuned, including enabling/disabling kvm or hax VM acceleration and alternate architectures for virtual machines.

Installing and running Windows 7:

First create your hard drive image and a 'windows7-data' volume to store disk images outside of the container.
```
docker run --rm -ti \
    -v windows7-data:/data \
    ksanislo/qemu \
    qemu-img create -f raw /data/sda.img 40G
```
Copy any install media you need into your newly created 'windows7-data' volume.

In this example, we're using Win7_Pro_SP1_English_COEM_x64.iso from the [official Microsoft website](https://www.microsoft.com/en-us/software-download/windows7) and the stable virtio-win_amd64.vfd [signed VirtIO drivers from RedHat](https://fedoraproject.org/wiki/Windows_Virtio_Drivers)
```
docker run --rm -ti \
    --privileged \
    --device /dev/kvm \
    -v /etc/resolv.conf:/etc/resolv.conf:ro \
    -v windows7-data:/data \
    -e SMP='8,cores=8,threads=1,sockets=1' \
    -e MEMORY='2G' \
    -e SDA='/data/sda.img,format=raw' \
    -e FDA='/data/virtio-win_amd64.vfd,format=raw' \
    -e CDA='/data/Win7_Pro_SP1_English_COEM_x64.iso' \
    -e BOOT="dc" \
    -e ENABLE_BRIDGE=Y \
    -e ENABLE_USBTABLET=Y \
    -p 5900:5900 \
    ksanislo/qemu
```
If everything worked, you should now be able to attach your [VNC viewer](https://www.realvnc.com/en/connect/download/viewer/) to vnc://\<docker-host-ip-address\>:5900, and install your new VM.

For windows, you will need to provide those VirtIO drivers from that virtual floppy before you can see the hard drives. Once the machine is up, make sure you install any other required drivers that you need and copy the contents of A:\ into the hard drive for later use.

Before you continue on, the following things are quite important:
* When first connecting to the virtual network the first time, you need to select the checkbox to treat all future networks as public.
* You have enabled Remote Desktop administration of the machine.
* You MUST enable Remote Desktop (TCP-In) for public networks in the Windows Firewall.

After the above has been prepared, you can switch over to macvlan inside and out. This mode gives the best throughput to other devices on the network, however you will be unable to talk with the VM from the docker host itself due to kernel filtering. 
Make sure you adjust this as needed for your LAN. Also, the internal DCHP server is disabled by these changes, so you will need to use your normal network DHCP server to assign a static IP for the virtual machine.
```
docker network create \
  -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range 192.168.1.32/28 \
  -o parent=br0 \
  macvlan
```
```
docker run -dt \
  --name windows7 \
  --network=macvlan \
  --privileged \
  --device /dev/kvm \
  -v windows7-data:/data \
  -e SMP='8,cores=8,threads=1,sockets=1' \
  -e MEMORY='8G' \
  -e SDA='/data/sda.img,format=raw' \
  ksanislo/qemu
```




