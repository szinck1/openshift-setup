## Setup Red Hat Atomic Host on VMware and Red Hat Satellite to host OpenShift Container Platform 3.6

### VMware Build
To build a new VM choose a custom VM and use the following specs:
  * 4 sockets 1 core
  * 32GB RAM
  * Attach to NIC to Network
  * Create a new disk of 80GB HDD


Edit virtual machine settings before poweron

  1. Remove Floppy drive
  2. Attach the cdrom ISO to the atomic installer on the datastore.
  3. Check "connect at power on"

Power up the VM

### OS Install
**Note:** Interupt the boot by pressing up arrow and select "Install Red Hat 
Enterprise Linux Atomic Host 7" and press tab


Right after quiet append the following:

```
net.ifnames=0 bios.devname=0
```

so it looks like

```
quiet net.ifnames=0 bios.devname=0
```

### Timezone
Once the GUI starts

* Click on Continue at the language selection
* Click on Date & Time

Set the City to **XYZ**

* click on Done

### Partitioning
Now let's setup the disk space

* Click on Installation Destination
* Click on "I will configure partitioning"
* Click on Done
* Click on "Click here to create them automatically"
* Click on the / partition

Change the Desired Capacity to **60 GiB**

* Click on swap
* Click on Done
* Click on Accept Changes

### Networking
* Click on Network & Hostname

Enter the Hostname

* click on Apply

* Click on Configure under eth0
* Click on IPv4 Settings
* Set Method to Manual

Enter the IP & DNS information

* Click on Save

* Click on the slider next to Ethernet so it shows **ON**

Confirm all the addresses are correct

* Click on Done

### Begin Installation
* Click on **Begin Installation**

* Click on **Root Password**
* Enter the default root one twice
* Click on Done

Wait for the install to complete 

* Click on "Reboot"

### VMware Post-Install
Once the reboot starts edit the VM in vSphere

* Click on CD/DVD drive 1
* Uncheck "connect at power on"
* Set the Device Type to "Host device"
* Click on OK


### OS Bootstrap

### Configure Chronyd time service

```
vi /etc/chrony.conf
```
* Remove/comment out default time servers
* Add your NTP server

```
systemctl restart chronyd
```

### Subscribe to Red Hat Satellite
```
curl -O http://satellite.server.tld/pub/katello-rhsm-consumer
chmod +x katello-rhsm-consumer
./katello-rhsm-consumer
```

Now run subscription manager

```
subscription-manager register --org="your_org" --activationkey="your_activation_key"
```

### Setup tooling
We need to setup open-vm-tools on VMware
```
atomic pull --storage=ostree registry.access.redhat.com/rhel7/open-vm-tools
atomic install --system registry.access.redhat.com/rhel7/open-vm-tools
```

Reboot the OS and we're done
```
atomic host upgrade
systemctl reboot
```
