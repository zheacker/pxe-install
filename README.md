# pxe-install
notes on setting up my PXE installer

# Outline of the PXE boot process
1. must install with 1GB of RAM (of virtualized hardware)
2. new host comes online, asks for DHCP service
3. DHCP (handled by dnsmasq on the router) responds with an IP address AND the location of a boot file (via the `dhcp-boot` option for dnsmasq) which is served up on a TFTP server on the PXE server machine
4. the new host now has its IP address, then loads the boot file from 2 (the file is `pxelinux.0`, located at **the root** of the TFTP server)
5. the pxelinux.0 boot file reads the pxe config file (called default in this simple setup). The default config file points to a kernel file and an initrd.img file, both located on our TFTP server (on Pixie)
6. I'm not 100% certain how the next few steps work, but I assume the kernel and initrd files (from step 4) are loaded as a temporary OS, and the default PXE config file points the new client to the FTP server, which contains all of the boot media from the .iso. The temporary OS then installs the OS from this FTP server.

# DD-WRT config
The router should already be configured with the LAN domain option (domainname.lan) and a static lease for the PXE server at 192.168.X.XXX

* tab: services > services
     * dnsmasq           =    enable
     * local dns         =    enable
     * no dns rebind     =    enable
     * additional dnsmasq options
          ```
          expand-hosts
          dhcp-boot=pxelinux.0,pixie.domainname.lan,192.168.X.XXX
          ```
          * **NOTE**: the dhcp-boot option does not seem to like absolute paths; it needs to be pointed at the TFTP server root folder (`/var/lib/tftpboot` in this case), and it only wants the filename in the boot option (`pxelinux.0`)
          * **NOTE 2**: I did not use the pxe-prompt or pxe-service options; don't fully understand how they mesh with the pxe options specified in the `pxelinux.cfg/default` file

# PXE server config
Now we'll setup the PXE server. This will be a small VM on the Dell box. We need to spin up the VM, install the pieces, and configure the TFTP server, as well as the PXE elements (like the boot list and the directory structure of the install media).

## general setup
*    install CentOS minimal
* `yum install epel-release`
* `yum update`
* `yum install vsftpd syslinux tftp-server wget xinetd vim`
* `setenforce 0`
* download CentOS DVD image with `wget`

## configure tftp server via xinetd
xinetd should be enabled but not active, will start on startup/reboot
* `vim /etc/xinetd.d/tftp`
```
service tftp
{
     socket type         =    dgram
     protocol            =    udp
     wait                =    yes
     user                =    root
     server              =    /usr/sbin/in.tftpd
     server_args         =    -s /var/lib/tftpboot
     disable             =    no
     per_source          =    11
     cps                 =    100 2
     flags               =    IPv4
}
```
* enable all-user access to tftpboot: `chmod 777 /var/lib/tftpboot`
* copy all syslinux file to the tftp server root directory: `cp -r /usr/share/syslinux/* /var/lib/tftpboot`
* each install option should have its own directory under `tftpboot/`: `mkdir /var/lib/tftpboot/centos7`
* a separate directory for the pxelinux config files: `mkdir /var/lib/tftpboot/pxelinux.cfg`
* copy `default` to `/var/lib/tftpboot/pxelinux.cfg/`
* mount the .iso: `mount -o loop /path/to/CentOS7Minimal.iso /mnt`
* copy the install media's pxeboot kernel and ramdisk to the appropriate tftpboot OS directory
     * `cp /mnt/images/pxeboot/vmlinuz /var/lib/tftpboot/centos7`
     * `cp /mnt/images/pxeboot/initrd.img /var/lib/tftpboot/centos7`
* copy all .iso contents to the ftp server location:
     * `cp -r /mnt/* /var/ftp/pub`
     * `chmod -R 755 /var/ftp/pub`
* `systemctl enable xinetd vsftpd`
* `systemctl restart xinetd vsftpd`
* `systemctl status xinetd vsftpd`
* copy kickstart file to FTP server:
     * `cp anaconda-ks.cfg /var/ftp/pub`
     * `chmod 755 /var/ftp/pub/anaconda-ks.cfg`

---
2 more commands I ran last time; pretty sure this is just a syntax validator for kickstart files

```
yum install pykickstart -y
ksvalidator /var/ftp/pub/anaconda-ks.cfg
```
---

* VNC password to watch installation is "Jarvis"
* new host's root password is my_num.Guy
* new host's hostname is renameme.domainname.lan
* zheacker user pw same as root

## things to do on the new host
1. rename host: `sudo hostnamectl set-hostname new_name`
2. `reboot`
3. change password
     * login as zheacker
     * `passwd`, change password
     * `exit`
     * login
4. `yum install epel-release`
5. `yum update`
