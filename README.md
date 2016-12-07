# pxe-install
notes on setting up my PXE installer

# Outline of the PXE boot process
1. must install with 1GB of RAM (of virtualized hardware)
2. host comes online, asks for DHCP service
3. DHCP (handled by DNSmasq on the router) responds with an IP address AND the location of a boot file (via the `dhcp-boot` option for DNSmasq) which is served up on a TFTP server on the PXE server machine
4. host has its IP address, then loads the boot file from 2 (the file is `pxelinux.0`, located at **the root** of the TFTP server)
5. 
