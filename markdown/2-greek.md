#It’s all Greek to Me
##Learning to Talk Like a Native


# IPMI
## Intelligent Power Management Interface
Note:
* IPMI provides:
* monitoring of hardware
* basic control of the hardware state
* and logging of deviations from expected values
* It is accessible via the system bus, and can be setup to respond via serial and LAN interfaces.


# BMC
## Baseboard Management Controller
Note:
* The BMC is the minimum management component
* As the heart of IPMI, this component provides the very basic power management and system status information.


# RMC
## Remote Management Cards
### DRACs, and iLOs, and RSAs, Oh my!
Note:
* Dell: DRAC (Dell Remote Access Controller)
* HP: iLO (intelligent Lights Out Adapter)
* IBM: RSA (Remote Supervisor Adapter)
* Provide remote shell and video consoles, hardware inventories and configuration


# CMDB
## Configuration Management Database

# DCIM
## Datacenter Information Management


# PXE & iPXE
## Preboot eXecution Environment
* PXE supports TFTP (Trivial File Transfer Protocol)
* iPXE supports HTTP, iSCSI, AoE, & FCoE
Note:
Intel introduced in 1999
iPXE (formally gPXE and Etherboot): An opensource implementation, capable over booting over a variety of network protocols
