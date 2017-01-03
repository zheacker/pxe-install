# pxe network install server
So you want to host a network installation server. Here's a quick walkthrough of how I set mine up. My PXE server is running on a CentOS 7 Minimal VM, and it serves up an image of CentOS 7 Minimal on the network (for my own little clone army of VMs). I'm also using a DD-WRT router with DNSmasq for DHCP requests.

There are lots of options for how to build this system out, particularly regarding how you serve the installation media itself (NFS, HTTP, FTP, etc.). I went with FTP, but there are plenty of guides online for other methods.

## Outline of the PXE boot process
1. A new host, our client, comes online and asks for DHCP service.
2. DNSmasq--on the router--responds with IP address AND the location of the TFTP server (on the PXE install server).
     - The dnsmasq option `dhcp-boot` provides this information
3. The client finds the TFTP server on the PXE server and locates `pxelinux.0` file at **the root** of the TFTP server.
     - The `pxelinux.0` boot loader is part of the syslinux package; syslinux must be installed and all contents of `/usr/share/syslinux/` copied to the root of the TFTP server.
4. `pxelinux.0` reads the `default` file from `pxelinux.cfg/`
5. `default` offers entries for each installable OS. The entries point to the appropriate kernel and initrd files on the TFTP server, and also to the appropriate kickstart file on the FTP server. (These entries also setup VNC to observe the install.)
6. The kernel and initrd files from the the TFTP server are loaded (as the temporary root file system to facilitate installation).
7. The kickstart file points to the `OS#_install_media` on the FTP server, and also manages the installation process.
8. That's it. The OS gets installed according to the process in the kickstart file and then you have an OS. Bam.

Example directory structures for TFTP and FTP servers:
```
# TFTP directory structure

/var/lib/tftpboot/
         |---pxelinux.0       # syslinux boot loader
         |---pxelinux.cfg/
         |   `--default       # pxelinux boot menu file
         |---OS1/
         |   |---vmlinuz      # kernel
         |   `---initrd.img   # ramdisk
         |---OS2/
             |---vmlinuz
             `---initrd.img
```

```
# FTP directory structure

/var/ftp/pub/
         |---OS1/
         |   |---OS1_kickstart
         |   `---OS1_install_media
         |---OS2/
             |---OS2_kickstart
             `---OS2_install_media
```

## DD-WRT config
My router is configured with the LAN domain option (domainname.lan) and a static lease for the PXE server at 192.168.X.XXX. To set these options, log into your router's web control panel:

* Under the services > services tab:
     * LAN Domain        =    domainname.lan
     * static leases:         assign your PXE host a static IP, let's say 192.168.4.20
     * dnsmasq           =    enable
     * local dns         =    enable
     * no dns rebind     =    enable
     * additional dnsmasq options
          ```
          expand-hosts
          dhcp-boot=pxelinux.0,pxehost.domainname.lan,192.168.4.20
          ```
          * **NOTE**: the dhcp-boot option does not like absolute paths; it expects the boot loader file (`pxelinux.0`) to be located at the root of the TFTP server at the given address
          * **NOTE 2**: I did not use the pxe-prompt or pxe-service options; don't fully understand how they mesh with the pxe options specified in the `pxelinux.cfg/default` file. If anyone cares to explain those, I'm all ears...

## PXE server config
Now we'll configure the PXE server. This is `pxehost.domainname.lan` at 192.168.4.20. We need to configure the TFTP server, as well as the PXE elements (like the boot list and the directory structure of the install media).

### general setup
Let's update the system and install all the pieces.

```
yum update
yum install tftp-server vsftpd syslinux xinetd wget vim
setenforce 0
wget -O /tmp/name.iso url.of.netinstall.iso
```

### configure tftp server via xinetd
Let's make sure xinetd and vsftpd are both up and running.
```
systemctl enable xinetd vsftpd
systemctl start xinetd vsftpd
```

Now we need to edit the TFTP config file. I'm using vim here.

`vim /etc/xinetd.d/tftp`

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
     * `cp anaconda-ks.cfg /var/ftp/pub/OSdir`
     * `chmod 755 /var/ftp/pub/OSdir/anaconda-ks.cfg`

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
