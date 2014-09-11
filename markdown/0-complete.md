![Sean P. Kane](images/me.jpg)

### Sean P. Kane - [@spkane](https://twitter.com/spkane)
### New Relic - Site Engineering
Note:
* Shutdown Chrome
* Silence Little Snitch
* Shutdown VPN Tint
* Shutdown Flux
* Bluetooth connect laptop to iPhone
* Start Unified Remote Control
* Full Screen Safari (Bookmarklet)
* Unified Remote Control (Web Slides)


## Automated Hardware Provisioning in the Real-world
#### or
### "How to Build Your Cloud Out of Metal"
Note:
* Print with something like: http://127.0.0.1:8080/?print-pdf#/


# - 0 to 60 -
* 00: Small hardware orders with manual server installs
* 60: Racks worth of hardware and automated installs

Note:
* We’ve all been there
* We studied, learned, and then built something new and interesting.
* People start using it, expectations increase, and suddenly what might have been a simple side-project is suddenly is become a foundation of something much bigger.


## What we need to build
![Provisioning Process](images/process.png)


# Design for Agility
## Just Enough Process

-
### What's the bare minimum we need to:
* Detect new hardware in a rack
* Automatically inventory and test it
* Automatically install and configure it


#It’s all Greek to Me
##Learning to Talk Like a Native


# IPMI
## Intelligent Power Management Interface
* Power Management
* Sensor Query & Tuning
* System Inventory & Logs
* SNMP Traps

Note:
* IPMI provides:
* monitoring of hardware
* basic control of the hardware state
* and logging of deviations from expected values
* It is accessible via the system bus, and can be setup to respond via serial and LAN interfaces.


# BMC
## Baseboard Management Controller
* The brain behind IPMI
* The BMC is the minimum management component


# RMC
## Remote Management Cards
### DRACs, and iLOs, and RSAs, Oh my!
* Always On Management
* SSH & Web Interfaces
* Deep system capabilities
* KVM over IP

Note:
* Dell: DRAC (Dell Remote Access Controller)
* HP: iLO (intelligent Lights Out Adapter)
* IBM: RSA (Remote Supervisor Adapter)
* Provide remote shell and video consoles, hardware inventories and configuration


### IPMI breakout
![IPMI Layout](images/ipmi-full.png)

Image: [Thomas-Krenn AG](http://www.thomas-krenn.com/en/wiki/File:Ipmi-schematische-darstellung.png)


# CMDB
## Configuration Management Database

# DCIM
## Datacenter Information Management


# Order & Rack
# the Hardware
* RMCs should be set to DHCP
* They can also use DDNS

-
#### DHCP - Dynamic Host Configuration Protocol
#### DDNS - Dynamic Domain Name Service
Note:
* RMCs obtain DHCP address
* RMCs register hostname & IP into DDNS


# Bootstrapping the System


## Waking up the RMC
![Provisioning Step 1](images/step-1.png)

Note:
* Making the first DHCP call


# DHCP trigger
## The magical bootstrap

```
  on commit {
      set clientip = binary-to-ascii(10, 8, ".", leased-address);
      set clientmac = binary-to-ascii(16, 8, ":", substring(hardware, 1, 6));
      set leasetime = binary-to-ascii(10,32,"",encode-int(lease-time,32));
      set vendorclass = pick-first-value(option vendor-class-identifier, "UNKNOWN");
      set ddns-hostname = pick-first-value(
          option fqdn.hostname,
          option host-name,
          concat( "dhcp-", binary-to-ascii(10, 8, "-",
          suffix(leased-address, 2))));
      execute("/usr/local/bin/dhcp-event.sh", "commit", clientip, vendorclass);
  }
```

* Note: The execute script is blocking. It should return immediately!


# Trigger Script

```bash
#!/bin/bash

API_USER_AUTH="something_secret"
API_HOST="http://cmdb.example.com"
API_BASE_URL="/api/v1"
API_DISCOVER_URL="/hosts/discover"
CURL_BIN="/usr/bin/curl"
URL="${API_HOST}${API_BASE_URL}${API_DISCOVER_URL}"

if [ $1 == "commit" ]; then
  ${CURL_BIN} --max-time 3 \
    --silent \
    -H 'Content-Type: application/json' \
    -X POST \
    --user ${API_USER_AUTH} \
    --data "{ \"host\": { \"ip_address\": \"${2}\", \"vclass\": \"${3}\" }}" \
    ${URL} > /dev/null
else
  exit 0
fi
```


## Detect and Test
![Provisioning Steps 2-3](images/steps-2-3.png)


# Interrogate the RMC
* ID system via Serial Number
* Determine hardware details


# Register the System
* Create a detailed entry in the CMDB/DCIM


# Boot the System
## PXE & iPXE
### Preboot eXecution Environment
* PXE supports TFTP (Trivial File Transfer Protocol)
* iPXE supports HTTP, iSCSI, AoE, FCoE & Wifi
* iPXE is easily bootable (media, pxe, etc.)
* iPXE is scriptable (logic trees, etc.)
* iPXE support extensions (menus, etc.)
Note:
Intel introduced in 1999
iPXE (formally gPXE and Etherboot): An opensource implementation, capable over booting over a variety of network protocols


# Chainloading iPXE from PXE
## Give me the Network Bootloader
```
  host dhcp-host10 {
    hardware ethernet FF:CA:3A:6E:66:00;
    fixed-address 10.20.250.42;
    option host-name "dhcp-host.example.com";
    option routers 10.20.250.1;
    if exists user-class and option user-class = "gPXE" {
        filename "http://10.20.250.10:80/ipxe/system/dhcp-host.example.com";
    } else if exists user-class and option user-class = "iPXE" {
        filename "http://10.20.250.10:80/ipxe/system/dhcp-host.example.com";
    } else {
        filename "undionly.kpxe";
    }
  }
```


# Booting iPXE
```
#!ipxe

:retry_dhcp
dhcp && isset http://boot.ipxe.org/demo/boot.php || goto retry_dhcp
echo Booting from http://boot.ipxe.org/demo/boot.php
chain http://boot.ipxe.org/demo/boot.php
```
Note:
* Create a retry_dhcp block
* Tries it until it gets a good boot file


# Booting the System
## Give me the Kernel & Ramdisk
```
#!ipxe

kernel vmlinuz-3.16.0-rc4 bootfile=http://boot.ipxe.org/demo/boot.php fastboot initrd=initrd.img
initrd initrd.img
boot
```
Note:
* The kernel & initrd images are downloaded from the same URL directory.


# Live Boot OS via iPXE
* Firmware updates
* Hardware Burn-in

Note:
* Time components are most likely to fail
* New firmware may cause problems
* Update all the firmware, before burning in the system


# Burnin Tools
* bonnie++: http://www.coker.com.au/bonnie++/
* memtester: http://pyropus.ca/software/memtester/
* Redhat Memtest: http://people.redhat.com/~dledford/memtest
* Phoronix Test Suite (PTS): http://www.phoronix-test-suite.com/


# Register the Results
* Update host record in CMDB/DCIM


## Installing Systems
![Provisioning Steps 4-5](images/steps-4-5.png)


# OS Installers
## Give me a script

-
### Kickstart or Preseed
* Kickstart example: http://goo.gl/z1fBwY
* Preseed example: http://goo.gl/QWXvqR


# Package Repositories
## Give me the media

-
### Yum, APT, etc.


# OS Configuration
## Give me the configuration

-
### Cloud-Init, Puppet, Chef, etc.
* Cloud-init examples: http://goo.gl/ogQUwN
* Puppet examples: http://goo.gl/Tk77SH
* Chef examples: http://goo.gl/wA5nDH


# Register the Results (again)
* Update host record in CMDB/DCIM


## Success!!
![Server Racks](images/racks.jpg)

Image: [Torkild Retvedt](https://flic.kr/p/6gYLHR)


## What we built
![Provisioning Process](images/process.png)


# Our Next Steps


# Micro-services
# Everywhere
### Small REST APIs
### for workflow construction
* Inventory (over-time)
* Health
* Remote configuration
* Etc.


# The Future...


# REST APIs
## in Hardware
* Software should not rely on SSH
* Humans should not use XML
* Nothing should implement a custom protocol


## Open Compute Project
### http://www.opencompute.org/


# Any Questions?

.
_________
### Interested in a great job?

.
#### Talk to me
#### *or visit*
#### http://newrelic.com/about/careers

.
### Sean P. Kane
### [@spkane](https://twitter.com/spkane)
