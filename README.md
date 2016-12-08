# pxe-install
notes on setting up my PXE installer

# Outline of the PXE boot process
1. must install with 1GB of RAM (of virtualized hardware)
2. host comes online, asks for DHCP service
3. DHCP (handled by dnsmasq on the router) responds with an IP address AND the location of a boot file (via the `dhcp-boot` option for dnsmasq) which is served up on a TFTP server on the PXE server machine
4. host has its IP address, then loads the boot file from 2 (the file is `pxelinux.0`, located at **the root** of the TFTP server)
5. the pxelinux.0 boot file reads the pxe config file (called default in this simple setup). The default config file points to a kernel file and an initrd.img file, both located on our TFTP server (on Pixie)
6. I'm not 100% certain how the next few steps work, but I assume the kernel and initrd files (from step 4) are loaded as a temporary OS, and the default PXE config file points the new client to the FTP server, which contains all of the boot media from the .iso. The temporary OS then installs the OS from this FTP server.

# DD-WRT config
* dnsmasq           =    enable
* local dns         =    enable
* no dns rebind     =    enable
* additional dnsmasq options
     * expand-hosts
     * dhcp-boot=pxelinux.0,pixie.stark.lan,192.168.1.141
          * **NOTE**: the dhcp-boot option does not seem to like absolute paths; it needs to be pointed at the TFTP server root folder (`/var/lib/tftpboot` in this case), and it only wants the filename in the boot option (`pxelinux.0`)
          * **NOTE 2**: I did not use the pxe-prompt or pxe-service options; don't fully understand how they mesh with the pxe options specified in the `pxelinux.cfg/default` file

# PXE server config
