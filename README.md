

# HOWTO PXE Boot C.H.I.P.

# Introduction

WORK-IN-PROGRESS

This is just a dump of the configuration files and commands used to get a functional CHIP PXE boot with 2 different NFS mounted  rootfs from 2 different NTC legacy stock images using the flashed stock uboot version and easisly switch between them. Any number of additional test rootfs can be used.

For now it's up to the reader to streamline all of this and make it work. 

This is work in progress and it is to serve only as a reference for future work. 
It may require adjustments to fit your particular environment.

## Context

- On the CHIP channel in Discord, user mfa298 suggested that booting the PocketCHIP via NFS/tftp via USB Ethernet. This way you could easily test different images. Even easier than booting via USB as you could change images and rootfs on the fly:
		   
		- USB Ethernet dongles are not likely to work as uboot would have to be compiled with support for whatever ethernet dongle to use.
	  
		- But it is not needed. The micro USB cable that powers the C.H.I.P. can be used as the stock uboot supports ethernet via USB. 

## Method Overview

- Setup tftp server
    - Add chip boot files to be served via tftp (vmlinuz/zImage, initrd, dtb, overlay, etc)
- Setup NFS
    - Prepare NFS share with legacy PocketCHIP image rootfs
    - NFS server: nfsd configuration
    - NFS client  (rootfs kernel command line parameters set in PXE config pxelinux.cfg)
- Setup DHCP
- Configure PXE
- Set uboot env vars on CHIP appropriately
- pxe boot in uboot prompt

# 0. Initial setup (Pre-Requisites)

- __IMPORTANT__: For now, the Direct UART serial pins (RX,TX,GND) on PocketCHIP with some kind of USB-to-Serial (UART) adapter on your computer (CH340 dongles, FlipperZero, etc) to access the uboot prompt
    - USB cannot be used as serial as it will serve as ethernet transport
    - As a serial terminal app you can use tio on linux or PuTTY on windows, for example.

- PocketCHIP with 4GB of Toshiba NAND
- legacy image flashed with uboot version running:
	-  `U-Boot SPL 2016.01-00115-g5f814bb (Dec 09 2016 - 23:00:24)`
- Server: Raspberry Pi 4 with GNU/Linux Debian 12


# 1. Server: Install Software (TFTP, DHCP & NFS)

```
sudo apt update
sudo apt install nfs-kernel-server tftpd-hpa isc-dhcp-server
```

# 2. Server: Setup USB Ethernet Networking

Create the USB ethernet device on the server

```
$ sudo nmcli con add type ethernet    \
  con-name ethernet-enxf8dc7a000001 \
  ifname "*" \
  802-3-ethernet.mac-address f8:dc:7a:00:00:01 \
  ipv4.method manual \
  ipv4.addresses 192.168.0.1/24 \
  ipv6.method ignore
```

```
$ sudo nmcli con modify ethernet-enxf8dc7a000001 connection.interface-name eth1
```

## Check USB Ethernet Networking

```
$ nmcli -f connection.interface-name,802-3-ethernet.mac-address con show ethernet-enxf8dc7a000001
```

```
connection.interface-name:              eth1
802-3-ethernet.mac-address:             F8:DC:7A:00:00:01
```
```
$ nmcli con
```
```
NAME                         UUID                                  TYPE      DEVICE
(...)
ethernet-enxf8dc7a000001     728950c4-2fa2-4e8c-874a-9ebea28e03dd  ethernet  --
(...)

```


# 3. Server: Setup DHCP service

## Configuration Files

### /etc/default/isc-dhcp-server

```
# Defaults for isc-dhcp-server (sourced by /etc/init.d/isc-dhcp-server)

# Path to dhcpd's config file (default: /etc/dhcp/dhcpd.conf).
DHCPDv4_CONF=/etc/dhcp/dhcpd.conf
#DHCPDv6_CONF=/etc/dhcp/dhcpd6.conf

# Path to dhcpd's PID file (default: /var/run/dhcpd.pid).
DHCPDv4_PID=/var/run/dhcpd.pid
#DHCPDv6_PID=/var/run/dhcpd6.pid

# Additional options to start dhcpd with.
#       Don't use options -cf or -pf here; use DHCPD_CONF/ DHCPD_PID instead
#OPTIONS=""

# On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
#       Separate multiple interfaces with spaces, e.g. "eth0 eth1".
INTERFACESv4="eth1"
INTERFACESv6=""
```

### /etc/dhcp/dhcpd.conf
```
option domain-name "attlocal.net";
option domain-name-servers 192.168.1.254, 8.8.8.8, 8.8.4.4, 1.1.1.1;

default-lease-time 600;
max-lease-time 7200;

ddns-update-style none;

subnet 192.168.0.0 netmask 255.255.255.0 {
    range 192.168.0.100 192.168.0.200;
    option routers 192.168.0.1;
    option subnet-mask 255.255.255.0;

    # PXE Specifics
    next-server 192.168.0.1;           # IP of your TFTP server
    filename "pxelinux.0";             # Boot file name for Legacy PXE
    # filename "bootx64.efi";          # Use this for UEFI instead

    # NFS Root Path (Optional, some kernels need this here)
    option root-path "192.168.0.1:/srv/pxe/rootfs";
}
```

### /etc/NetworkManager/dispatcher.d/99-dhcp-server

A script is needed to restart the dhcpd service when usb ethernet device eth1 is plugged and unplugged

```
#!/bin/bash

INTERFACE=$1
ACTION=$2

if [ "$INTERFACE" = "eth1" ]; then
    case "$ACTION" in
        up)
            # The interface is up and has an IP, now safe to start/restart
            systemctl restart isc-dhcp-server
            ;;
        down)
            systemctl stop isc-dhcp-server
            ;;
    esac
fi
```

Add execution permissions
```
sudo chmod +x /etc/NetworkManager/dispatcher.d/99-dhcp-server
```


## Apply DHCP configuration

```
sudo systemctl restart isc-dhcp-server
```


# 4. Server: Setup NFS service

## Setup the bootable C.H.I.P. root filesystems

As an example the root filesystems extracted from the original NTC stock images were used

- stable-gui-b149.tar.gz
- stable-pocketchip-b126.tar.gz

These archives are available in archive.org as a torrent: [C.h.i.p.FlashCollection-As-tar.gz-Archives_archive.torrent](https://archive.org/download/C.h.i.p.FlashCollection-As-tar.gz-Archives/C.h.i.p.FlashCollection-As-tar.gz-Archives_archive.torrent)

You can download it using any torrent client like transmission-cli

```
transmission-cli https://archive.org/download/C.h.i.p.FlashCollection-As-tar.gz-Archives/C.h.i.p.FlashCollection-As-tar.gz-Archives_archive.torrent
```

Extract both archives into the nfs exported directories:

```
tar xfvz stable-gui-b149.tar.gz -C /srv/nfs/rootfs/b149  
tar xfvz stable-pocketchip-b126.tar.gz -C /srv/nfs/rootfs/b149 
```

## Configure the NFS shares - /etc/exports
```
# /etc/exports
/srv/nfs/rootfs/b126  192.168.0.0/24(rw,sync,no_subtree_check,no_root_squash)
/srv/nfs/rootfs/b149  192.168.0.0/24(rw,sync,no_subtree_check,no_root_squash)
```

Apply commands:
```
sudo exportfs -ra
sudo systemctl enable rpcbind nfs-server
sudo systemctl restart rpcbind nfs-server
```

# 5. Server: PXE configuration

## NOTE: pxelinux.cfg used but boot.scr maybe  better

- For the PXE boot, a pxelinux.cfg based configuration is being used to specify the kernel boot files
- However the original NTC uboot version does not support the pxeboot.cfg devicetree-overlay command so the overlay file must be loaded via tftp to memory before the pxe boot is initiated
- For better consistency and flexibility a better approach could be loading a boot script via tftp and execute it

## /srv/tftp/pxelinux.cfg/default

```
menu include pxelinux.cfg/menus/base.menu
timeout 500

default b126-stable-pocketchip
```

## /srv/tftp/pxelinux.cfg/menus/base.menu

```
menu title Boot from which CHIP image?

# Original PocketCHIP image
label b126-stable-pocketchip
    menu label Original PocketCHIP image (stable-pocketchip-b126)
    kernel pxe/b126/zImage
    fdt pxe/b126/sun5i-r8-chip.dtb
    initrd pxe/b126/initrd.uimage
    append rw earlyprintk root=/dev/nfs nfsroot=192.168.0.1:/srv/pxe/rootfs/b126 g_cdc.dev_addr=f8:dc:7a:00:00:02 g_cdc.host_addr=f8:dc:7a:00:00:01

# Original PocketCHIP GUI image
label b149-stable-gui
    menu label Original PocketCHIP GUI image (stable-gui-b149)
    kernel pxe/b149/zImage
    fdt pxe/b149/sun5i-r8-chip.dtb
    initrd pxe/b149/initrd.uimage
    append rw earlyprintk root=/dev/nfs nfsroot=192.168.0.1:/srv/pxe/rootfs/b149 g_cdc.dev_addr=f8:dc:7a:00:00:02 g_cdc.host_addr=f8:dc:7a:00:00:01

```

## PXE Files and Directories Layout 



```
/srv/pxe/b126 -> boot/chip/stable-pocketchip-b126/
/srv/pxe/b149 -> boot/chip/stable-gui-b149/

/srv/pxe/boot/chip/stable-gui-b149
/srv/pxe/boot/chip/stable-gui-b149/dip-9d011a-1.dtbo -> ../../../rootfs/b149/lib/firmware/nextthingco/chip/early/dip-9d011a-1.dtbo
/srv/pxe/boot/chip/stable-gui-b149/initrd.uimage -> ../../../rootfs/b149/boot/initrd.img-4.4.13-ntc-mlc
/srv/pxe/boot/chip/stable-gui-b149/sun5i-r8-chip.dtb -> ../../../rootfs/b149/boot/sun5i-r8-chip.dtb
/srv/pxe/boot/chip/stable-gui-b149/zImage -> ../../../rootfs/b149/boot/zImage

/srv/pxe/boot/chip/stable-pocketchip-b126
/srv/pxe/boot/chip/stable-pocketchip-b126/dip-9d011a-1.dtbo -> ../../../rootfs/b126/lib/firmware/nextthingco/chip/early/dip-9d011a-1.dtbo
/srv/pxe/boot/chip/stable-pocketchip-b126/initrd.uimage -> ../../../rootfs/b126/boot/initrd.uimage
/srv/pxe/boot/chip/stable-pocketchip-b126/sun5i-r8-chip.dtb -> ../../../rootfs/b126/boot/sun5i-r8-chip.dtb
/srv/pxe/boot/chip/stable-pocketchip-b126/zImage -> ../../../rootfs/b126/boot/zImage

/srv/pxelinux.cfg/55dee692-14be-11f1-afe8-8fde39c692f2 -> default
/srv/pxelinux.cfg/default
/srv/pxelinux.cfg/menus/base.menu

```

# 6. CHIP: Setup UBoot ENV (uboot prompt - UART serial console required)

```
# ipaddr: IP address of the board
# serverip: IP address of your PC or server
# ethact: controls which interface is currently active.
# usbnet_devaddr: MAC address on the device side
# usbnet_hostaddr: MAC address on the host side
# pxeuuid: An identifier to one-shot match the pxelinux.cfg 
#          config file  (check pxelinux.cfg directory)
# --
#
# dip_overlay_cmd: the command to load the overlay file
# dip_overlay_dir: the directory where the overlay file is at
# ^
# '-- Since this version of uboot does not support the devicetree-overlay 
# label while loading the pxelinux.cfg file, it cannot be configured 
# in the pxelinux.cfg file.
#
# As such, it is preloaded via tftp and the same one file from b126 
# is statically configured and used. For PocketCHIP images it should 
# be OK to use the same file across all rootfs but if a different 
# overlay file needs to be used, it has to be manually preloaded or
# some kind of automated loading of the overlay will have to be devised 
#
# --

setenv ipaddr 192.168.0.100
setenv serverip 192.168.0.1
setenv ethact usb_ether
setenv usbnet_devaddr f8:dc:7a:00:00:02
setenv usbnet_hostaddr f8:dc:7a:00:00:01
setenv pxeuuid 55dee692-14be-11f1-afe8-8fde39c692f2

setenv dip_overlay_name dip-9d011a-1.dtbo
setenv dip_overlay_dir chip/b126
setenv dip_overlay_cmd 'if test -n "${dip_overlay_name}"; then tftp ${dip_addr_r} ${dip_overlay_dir}/${dip_overlay_name}; fi'

setenv rootpath=192.168.0.1:/srv/nfs/rootfs
setenv gatewayip=192.168.0.1
setenv dip_overlay_cmd=if test -n "${dip_overlay_name}"; then tftp ${dip_addr_r} ${dip_overlay_dir}/${dip_overlay_name}; fi
setenv dip_overlay_dir=pxe/b126
setenv pxeuuid=55dee692-14be-11f1-afe8-8fde39c692f2

saveenv
```


### printenv: Main uboot env variables

```
rootpath=192.168.0.1:/srv/nfs/rootfs
gatewayip=192.168.0.1
ipaddr=192.168.0.100
dip_overlay_cmd=if test -n "${dip_overlay_name}"; then tftp ${dip_addr_r} ${dip_overlay_dir}/${dip_overlay_name}; fi
dip_overlay_dir=pxe/b126
ethact=usb_ether
ethaddr=02:94:08:c0:e1:b2
pxeuuid=55dee692-14be-11f1-afe8-8fde39c692f2
usbnet_devaddr=f8:dc:7a:00:00:02
usbnet_hostaddr=f8:dc:7a:00:00:01

```

### printenv: other vars (not changed)

```
baudrate=115200
boot_a_script=load ${devtype} ${devnum}:${distro_bootpart} ${scriptaddr} ${prefix}${script}; source ${scriptaddr}
boot_extlinux=sysboot ${devtype} ${devnum}:${distro_bootpart} any ${scriptaddr} ${prefix}extlinux/extlinux.conf
boot_initrd=mtdparts; ubi part UBI; ubifsmount ubi0:rootfs; ubifsload $fdt_addr_r /boot/sun5i-r8-chip.dtb; ubifsload 0x44000000 /boot/initrd.uimage; ubifsload $kernel_addr_r /boot/zImage; bootz $kernel_addr_r 0x44000000 $fdt_addr_r
boot_noinitrd=mtdparts; ubi part UBI; ubifsmount ubi0:rootfs; ubifsload $fdt_addr_r /boot/sun5i-r8-chip.dtb; ubifsload $kernel_addr_r /boot/zImage; bootz $kernel_addr_r - $fdt_addr_r
boot_prefixes=/ /boot/
boot_script_dhcp=boot.scr.uimg
boot_scripts=boot.scr.uimg boot.scr
boot_targets=fel usb0 pxe dhcp
bootargs=root=ubi0:rootfs rootfstype=ubifs rw earlyprintk ubi.mtd=4
bootcmd=run test_fastboot; if test -n ${fel_booted} && test -n ${scriptaddr}; then echo (FEL boot); source ${scriptaddr}; fi; for path in ${bootpaths}; do run boot_$path; done
bootcmd_dhcp=usb start; if dhcp ${scriptaddr} ${boot_script_dhcp}; then source ${scriptaddr}; fi
bootcmd_fel=if test -n ${fel_booted} && test -n ${scriptaddr}; then echo '(FEL boot)'; source ${scriptaddr}; fi
bootcmd_pxe=usb start; dhcp; if pxe get; then pxe boot; fi
bootcmd_usb0=setenv devnum 0; run usb_boot
bootdelay=2
bootfile=pxelinux.0
bootm_size=0xa000000
bootpaths=initrd noinitrd
console=ttyS0,115200
dfu_alt_info_ram=kernel ram 0x42000000 0x1000000;fdt ram 0x43000000 0x100000;ramdisk ram 0x43300000 0x4000000
dip_addr_r=0x43400000
distro_bootcmd=for target in ${boot_targets}; do run bootcmd_${target}; done
fdt_addr_r=0x43000000
fdtcontroladdr=5ab25520
fdtfile=sun5i-r8-chip.dtb
fileaddr=43200000
filesize=71
kernel_addr_r=0x42000000
mtdids=nand0=sunxi-nand.0
mtdparts=mtdparts=sunxi-nand.0:4m(spl),4m(spl-backup),4m(uboot),4m(env),-(UBI)
netmask=255.255.255.0
preboot=usb start
pxefile_addr_r=0x43200000
ramdisk_addr_r=0x43300000
scan_dev_for_boot=echo Scanning ${devtype} ${devnum}:${distro_bootpart}...; for prefix in ${boot_prefixes}; do run scan_dev_for_extlinux; run scan_dev_for_scripts; done
scan_dev_for_boot_part=part list ${devtype} ${devnum} -bootable devplist; env exists devplist || setenv devplist 1; for distro_bootpart in ${devplist}; do if fstype ${devtype} ${devnum}:${distro_bootpart} bootfstype; then run scan_dev_for_boot; fi; done
scan_dev_for_extlinux=if test -e ${devtype} ${devnum}:${distro_bootpart} ${prefix}extlinux/extlinux.conf; then echo Found ${prefix}extlinux/extlinux.conf; run boot_extlinux; echo SCRIPT FAILED: continuing...; fi
scan_dev_for_scripts=for script in ${boot_scripts}; do if test -e ${devtype} ${devnum}:${distro_bootpart} ${prefix}${script}; then echo Found U-Boot script ${prefix}${script}; run boot_a_script; echo SCRIPT FAILED: continuing...; fi; done
scriptaddr=0x43100000
serial#=1625429408c0e1b2
serverip=192.168.0.1
splashpos=m,m
stderr=serial
stdin=serial,usbkbd
stdout=serial
ubifs_boot=if ubi part UBI && ubifsmount ubi${devnum}:boot; then setenv devtype ubi; setenv bootpart 0; run scan_dev_for_boot; fi
usb_boot=usb start; if usb dev ${devnum}; then setenv devtype usb; run scan_dev_for_boot_part; fi
video-mode=sunxi:480x272-16@60,monitor=lcd

```

# 7. PXE BOOT IT!!

In the UBoot Prompt - via the UART Serial Terminal (press space right when CHIP starts booting )

```
=> pxe get
```
```
=> pxe boot
```


# A. APPENDIX - Small tftp test to check that USB networking works on the original NTC Uboot
### Commands

```
# -- UBOOT prompt: (via CHIP UART serial press any key to stop uboot at start)

# ipaddr: IP address of the board
# serverip: IP address of your PC or server
# ethact: controls which interface is currently active.
# usbnet_devaddr: MAC address on the device side
# usbnet_hostaddr: MAC address on the host side

setenv ipaddr 192.168.0.100
setenv serverip 192.168.0.1
setenv ethact usb_ether
setenv usbnet_devaddr f8:dc:7a:00:00:02
setenv usbnet_hostaddr f8:dc:7a:00:00:01
saveenv

# -- On linux (tftp server)

sudo nmcli con add type ethernet \
  con-name ethernet-enxf8dc7a000001 \
  ifname "*" \
  802-3-ethernet.mac-address f8:dc:7a:00:00:01 \
  ipv4.method manual \
  ipv4.addresses 192.168.0.1/24 \
  ipv6.method ignore
sudo apt install tftpd-hpa
echo "Hello World!!" | sudo cat > /srv/tftp/hello.txt

# -- UBOOT prompt
tftp 0x81000000 textfile.txt
md 0x81000000

```

### UBOOT Serial OUTPUT:

```
Hit any key to stop autoboot:  0

=> setenv ipaddr 192.168.0.100
=> setenv serverip 192.168.0.1
=> setenv ethact usb_ether
=> setenv usbnet_devaddr f8:dc:7a:00:00:02
=> setenv usbnet_hostaddr f8:dc:7a:00:00:01
=> saveenv

Saving Environment to NAND...
Erasing NAND...
Erasing at 0xc00000 -- 100% complete.
Writing to NAND... OK

=> ping 192.168.0.1

using musb-hdrc, OUT ep1out IN ep1in STATUS ep2in
MAC f8:dc:7a:00:00:02
HOST MAC f8:dc:7a:00:00:01
RNDIS ready
high speed config #2: 2 mA, Ethernet Gadget, using RNDIS
USB RNDIS network up!
Using usb_ether device
host 192.168.0.1 is alive

=> tftp 0x81000000 hello.txt

using musb-hdrc, OUT ep1out IN ep1in STATUS ep2in
MAC f8:dc:7a:00:00:02
HOST MAC f8:dc:7a:00:00:01
RNDIS ready
high speed config #2: 2 mA, Ethernet Gadget, using RNDIS
USB RNDIS network up!
Using usb_ether device
TFTP from server 192.168.0.1; our IP address is 192.168.0.100
Filename 'hello.txt'.
Load address: 0x81000000
Loading: #
         0 Bytes/s
done
Bytes transferred = 14 (e hex)

=> md 0x81000000

81000000: 6c6c6548 6f57206f 21646c72 ffff0a21    Hello World!!...
81000010: ffffffff ffeffbbf d7bfbfbe ffefdfbf    ................
81000020: ffffffff 5fe6ff6b fdf7feff f6fffbff    ....k.._........
81000030: ffffffff bfffcbcb fffffbdc ffffffff    ................
81000040: ffffffdf 7fef7ffd ffffb5ff ffffefff    ................
81000050: fff7ffff ff7dfbdb fff3bd9f ffffedff    ......}.........
81000060: ffffffff fe7bfa7f fffeb75f ffffffff    ......{._.......
81000070: ffdffbff ff7fbffd fbffffff fffffffb    ................
81000080: ffffff7f ffeeffff ffffa4df ffffffff    ................
81000090: fffffeff ffbffffa ffdfffff ffffffff    ................
810000a0: ffffffff beffdfff fffefff7 ffffffff    ................
810000b0: ffffffff efefb9fb ffdffae6 fefffeff    ................
810000c0: fff7ffff ffffabff bfffefff ffffdebf    ................
810000d0: fff77bff dffffe97 efffe9df ffffffef    .{..............
810000e0: ffffffef ffffffff dfbfde7f dfffffff    ................
810000f0: ffffffff efffff7d ddfffffc ffffdfff    ....}...........

```


# Troubleshooting

```
$ nmcli -f connection.interface-name,802-3-ethernet.mac-address con show ethernet-enxf8dc7a000001
```

```
connection.interface-name:              eth1
802-3-ethernet.mac-address:             F8:DC:7A:00:00:01

pi@pihole:~ $ nmcli con

NAME                         UUID                                  TYPE      DEVICE
ethernet-enxf8dc7a000001     728950c4-2fa2-4e8c-874a-9ebea28e03dd  ethernet  --
Wired connection 1           69fe6084-3197-3858-992c-9d25f59a2b64  ethernet  --

```


# REFERENCES:

- [Use U-Boot PXE Boot and extlinux.conf](https://docs.u-boot.org/en/latest/usage/pxe.html)
- [How to configure a Raspberry Pi as a PXE boot server](https://linuxconfig.org/how-to-configure-a-raspberry-pi-as-a-pxe-boot-server)


