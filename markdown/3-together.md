# Bootstrapping the System


# Discovering the RMC
## "Hello? Is anybody out there?"
Note:
* RAC pre-configured to DHCP


# DHCP
## Dynamic Host Configuration Protocol

# DDNS
## Dynamic Domain Name Service
Note:
* RMCs obtain DHCP address
* RMCs register hostname & IP into DDNS


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


# Interrogate the RMC
* ID system via Serial Number
* Determine hardware details


# Register the System
* Create a detailed entry in the CMDB/DCIM


# PXE boot the System


# Bootstrapping iPXE from PXE
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
```
#!ipxe

kernel vmlinuz-3.16.0-rc4 bootfile=http://boot.ipxe.org/demo/boot.php fastboot initrd=initrd.img
initrd initrd.img
boot
```
Note:
* The kernel & initrd images are downloaded from the same URL directory.
