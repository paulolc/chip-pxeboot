
# HOWTO PXE Boot C.H.I.P.

# Introduction

WORK-IN-PROGRESS

This is just a dump of the configuration files and commands used to get a functional CHIP PXE boot with 2 different NFS mounted  rootfs from 2 different NTC legacy stock images using the flashed stock uboot version and easily switch between them. Any number of additional test rootfs can be used.

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
tar xfvz stable-pocketchip-b126.tar.gz -C /srv/nfs/rootfs/b126
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

default b126
```

## /srv/tftp/pxelinux.cfg/menus/base.menu

```
menu title Boot from which CHIP image?

# Original PocketCHIP image
label b126
    menu label Original PocketCHIP image (stable-pocketchip-b126)
    kernel pxe/b126/zImage
    fdt pxe/b126/sun5i-r8-chip.dtb
    initrd pxe/b126/initrd.uimage
    append rw earlyprintk root=/dev/nfs nfsroot=192.168.0.1:/srv/pxe/rootfs/b126 g_cdc.dev_addr=f8:dc:7a:00:00:02 g_cdc.host_addr=f8:dc:7a:00:00:01

# Original PocketCHIP GUI image
label b149
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

# APPENDIX B. Full PXE boot serial output
```
U-Boot SPL 2016.01-00115-g5f814bb (Dec 09 2016 - 23:00:24)
Fuel Gauge: 78%
Battery Voltage: 3792mV
DRAM: 512 MiB
CPU: 1008000000Hz, AXI/AHB/APB: 3/2/2
Trying to boot from NAND


U-Boot 2016.01-00115-g5f814bb (Dec 09 2016 - 23:00:24 +0000) Allwinner Technology

CPU:   Allwinner A13 (SUN5I)
I2C:   ready
DRAM:  512 MiB
NAND:  4096 MiB
DIP: Found PocketCHIP (0x1) from Next Thing Co. (0x9d011a) detected
video-mode 480x272-16@60 not available, falling back to 1024x768-24@60
Setting up a 480x272 lcd console (overscan 0x0)
In:    serial
Out:   serial
Err:   serial
Net:   usb_ether
starting USB...
No controllers found
Hit any key to stop autoboot:  0
```

### Execute UBoot PXE get to get pxelinux.cfg file 

pxelinux.cfg/55dee692-14be-11f1-afe8-8fde39c692f2 - pxeuuid uboot var

```
=> pxe get
```
```
Retrieving file: pxelinux.cfg/55dee692-14be-11f1-afe8-8fde39c692f2
using musb-hdrc, OUT ep1out IN ep1in STATUS ep2in
MAC f8:dc:7a:00:00:02
HOST MAC f8:dc:7a:00:00:01
RNDIS ready
musb-hdrc: peripheral reset irq lost!
high speed config #2: 2 mA, Ethernet Gadget, using RNDIS
USB RNDIS network up!
Using usb_ether device
TFTP from server 192.168.0.1; our IP address is 192.168.0.100
Filename 'pxelinux.cfg/55dee692-14be-11f1-afe8-8fde39c692f2'.
Load address: 0x43200000
Loading: #
         0 Bytes/s
done
Bytes transferred = 113 (71 hex)
Config file found
```
### Execute UBoot PXE Boot

The UBoot pxe boot command loads all kernel files according to pxelinux config 

```
=> pxe boot
```
```
Retrieving file: pxelinux.cfg/menus/base.menu
using musb-hdrc, OUT ep1out IN ep1in STATUS ep2in
MAC f8:dc:7a:00:00:02
HOST MAC f8:dc:7a:00:00:01
RNDIS ready
high speed config #2: 2 mA, Ethernet Gadget, using RNDIS
USB RNDIS network up!
Using usb_ether device
TFTP from server 192.168.0.1; our IP address is 192.168.0.100
Filename 'pxelinux.cfg/menus/base.menu'.
Load address: 0x43200074
Loading: #
         375 KiB/s
done
Bytes transferred = 769 (301 hex)
```
### Choose the rootfs via the configured PXE Menu 
(see PXE config section above)

```
Boot from which CHIP image?
1:      Original PocketCHIP image (stable-pocketchip-b126)
2:      Original PocketCHIP GUI image (stable-gui-b149)
Enter choice: 1
```
### Load All Kernel files (output)

```
1:      Original PocketCHIP image (stable-pocketchip-b126)
Retrieving file: pxe/b126/initrd.uimage
using musb-hdrc, OUT ep1out IN ep1in STATUS ep2in
MAC f8:dc:7a:00:00:02
HOST MAC f8:dc:7a:00:00:01
RNDIS ready
high speed config #2: 2 mA, Ethernet Gadget, using RNDIS
USB RNDIS network up!
Using usb_ether device
TFTP from server 192.168.0.1; our IP address is 192.168.0.100
Filename 'pxe/b126/initrd.uimage'.
Load address: 0x43300000
Loading: #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         ####
         5.5 MiB/s
done
Bytes transferred = 8640494 (83d7ee hex)
Retrieving file: pxe/b126/zImage
using musb-hdrc, OUT ep1out IN ep1in STATUS ep2in
MAC f8:dc:7a:00:00:02
HOST MAC f8:dc:7a:00:00:01
RNDIS ready
high speed config #2: 2 mA, Ethernet Gadget, using RNDIS
USB RNDIS network up!
Using usb_ether device
TFTP from server 192.168.0.1; our IP address is 192.168.0.100
Filename 'pxe/b126/zImage'.
Load address: 0x42000000
Loading: #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #############################
         5.5 MiB/s
done
Bytes transferred = 7097288 (6c4bc8 hex)
append: rw earlyprintk root=/dev/nfs nfsroot=192.168.0.1:/srv/nfs/rootfs/b126 g_cdc.dev_addr=f8:dc:7a:00:00:02 g_cdc.host_addr=f8:dc:7a:00:00:01
Retrieving file: pxe/b126/sun5i-r8-chip.dtb
using musb-hdrc, OUT ep1out IN ep1in STATUS ep2in
MAC f8:dc:7a:00:00:02
HOST MAC f8:dc:7a:00:00:01
RNDIS ready
high speed config #2: 2 mA, Ethernet Gadget, using RNDIS
USB RNDIS network up!
Using usb_ether device
TFTP from server 192.168.0.1; our IP address is 192.168.0.100
Filename 'pxe/b126/sun5i-r8-chip.dtb'.
Load address: 0x43000000
Loading: ###
         4.3 MiB/s
done
Bytes transferred = 40432 (9df0 hex)
Kernel image @ 0x42000000 [ 0x000000 - 0x6c4bc8 ]
## Loading init Ramdisk from Legacy Image at 43300000 ...
   Image Name:
   Image Type:   ARM Linux RAMDisk Image (uncompressed)
   Data Size:    8640430 Bytes = 8.2 MiB
   Load Address: 00000000
   Entry Point:  00000000
   Verifying Checksum ... OK
## Flattened Device Tree blob at 43000000
   Booting using the fdt blob at 0x43000000
   Loading Ramdisk to 497c2000, end 49fff7ae ... OK
   Loading Device Tree to 497b5000, end 497c1def ... OK
DIP: Applying dip overlay dip-9d011a-1.dtbo
using musb-hdrc, OUT ep1out IN ep1in STATUS ep2in
MAC f8:dc:7a:00:00:02
HOST MAC f8:dc:7a:00:00:01
RNDIS ready
high speed config #2: 2 mA, Ethernet Gadget, using RNDIS
USB RNDIS network up!
Using usb_ether device
TFTP from server 192.168.0.1; our IP address is 192.168.0.100
Filename 'pxe/b126/dip-9d011a-1.dtbo'.
Load address: 0x43400000
Loading: #
         1.2 MiB/s
done
Bytes transferred = 3741 (e9d hex)
```

### Kernel starting... (output)

```
Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Initializing cgroup subsys cpuset
[    0.000000] Initializing cgroup subsys cpu
[    0.000000] Initializing cgroup subsys cpuacct
[    0.000000] Linux version 4.4.13-ntc-mlc (bamboo@ip-172-31-21-118) (gcc version 5.2.1 20151010 (Ubuntu 5.2.1-22ubuntu1) ) #1 SMP Tue Dec 6 21:38:00 UTC 2016
[    0.000000] CPU: ARMv7 Processor [413fc082] revision 2 (ARMv7), cr=10c5387d
[    0.000000] CPU: PIPT / VIPT nonaliasing data cache, VIPT aliasing instruction cache
[    0.000000] Machine model: NextThing C.H.I.P.
[    0.000000] cma: Reserved 64 MiB at 0x5b800000
[    0.000000] Memory policy: Data cache writeback
[    0.000000] CPU: All CPU(s) started in SVC mode.
[    0.000000] PERCPU: Embedded 12 pages/cpu @dff2a000 s19008 r8192 d21952 u49152
[    0.000000] Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 129793
[    0.000000] Kernel command line: rw earlyprintk root=/dev/nfs nfsroot=192.168.0.1:/srv/nfs/rootfs/b126 g_cdc.dev_addr=f8:dc:7a:00:00:02 g_cdc.host_addr=f8:dc:7a:00:00:01
[    0.000000] PID hash table entries: 2048 (order: 1, 8192 bytes)
[    0.000000] Dentry cache hash table entries: 65536 (order: 6, 262144 bytes)
[    0.000000] Inode-cache hash table entries: 32768 (order: 5, 131072 bytes)
[    0.000000] Memory: 428536K/523776K available (9094K kernel code, 1140K rwdata, 4192K rodata, 1068K init, 349K bss, 29704K reserved, 65536K cma-reserved, 0K highmem)
[    0.000000] Virtual kernel memory layout:
[    0.000000]     vector  : 0xffff0000 - 0xffff1000   (   4 kB)
[    0.000000]     fixmap  : 0xffc00000 - 0xfff00000   (3072 kB)
[    0.000000]     vmalloc : 0xe0000000 - 0xff800000   ( 504 MB)
[    0.000000]     lowmem  : 0xc0000000 - 0xdff80000   ( 511 MB)
[    0.000000]     pkmap   : 0xbfe00000 - 0xc0000000   (   2 MB)
[    0.000000]     modules : 0xbf000000 - 0xbfe00000   (  14 MB)
[    0.000000]       .text : 0xc0208000 - 0xc0f02804   (13291 kB)
[    0.000000]       .init : 0xc0f03000 - 0xc100e000   (1068 kB)
[    0.000000]       .data : 0xc100e000 - 0xc112b008   (1141 kB)
[    0.000000]        .bss : 0xc112e000 - 0xc1185740   ( 350 kB)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
[    0.000000] Hierarchical RCU implementation.
[    0.000000]  Build-time adjustment of leaf fanout to 32.
[    0.000000]  RCU restricting CPUs from NR_CPUS=16 to nr_cpu_ids=1.
[    0.000000] RCU: Adjusting geometry for rcu_fanout_leaf=32, nr_cpu_ids=1
[    0.000000] NR_IRQS:16 nr_irqs:16 16
[    0.000000] clocksource: timer: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 79635851949 ns
[    0.000000] clocksource: timer: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 6370868154 ns
[    0.000000] sched_clock: 32 bits at 200 Hz, resolution 5000000ns, wraps every 10737418237500000ns
[    0.000000] Console: colour dummy device 80x30
[    0.000000] console [tty0] enabled
[    0.025000] Calibrating delay loop... 1002.70 BogoMIPS (lpj=2506752)
[    0.025000] pid_max: default: 32768 minimum: 301
[    0.025000] Mount-cache hash table entries: 1024 (order: 0, 4096 bytes)
[    0.025000] Mountpoint-cache hash table entries: 1024 (order: 0, 4096 bytes)
[    0.025000] Initializing cgroup subsys io
[    0.025000] Initializing cgroup subsys memory
[    0.025000] Initializing cgroup subsys devices
[    0.025000] Initializing cgroup subsys freezer
[    0.025000] Initializing cgroup subsys net_cls
[    0.025000] Initializing cgroup subsys net_prio
[    0.025000] Initializing cgroup subsys pids
[    0.025000] CPU: Testing write buffer coherency: ok
[    0.025000] CPU0: thread -1, cpu 0, socket -1, mpidr 0
[    0.025000] Setting up static identity map for 0x40209000 - 0x40209098
[    0.030000] Brought up 1 CPUs
[    0.030000] SMP: Total of 1 processors activated (1002.70 BogoMIPS).
[    0.030000] CPU: All CPU(s) started in SVC mode.
[    0.030000] devtmpfs: initialized
[    0.045000] VFP support v0.3: implementor 41 architecture 3 part 30 variant c rev 3
[    0.045000] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 9556302231375000 ns
[    0.050000] pinctrl core: initialized pinctrl subsystem
[    0.050000] NET: Registered protocol family 16
[    0.055000] DMA: preallocated 256 KiB pool for atomic coherent allocations
[    0.055000] cpuidle: using governor ladder
[    0.055000] cpuidle: using governor menu
[    0.065000] No ATAGs?
[    0.065000] hw-breakpoint: debug architecture 0x4 unsupported.
[    0.065000] Serial: AMBA PL011 UART driver
[    0.080000] reg-fixed-voltage usb0-vbus: could not find pctldev for node /soc@01c00000/pinctrl@01c20800/chip_vbus_pin@0, deferring probe
[    0.085000] vgaarb: loaded
[    0.085000] SCSI subsystem initialized
[    0.085000] usbcore: registered new interface driver usbfs
[    0.085000] usbcore: registered new interface driver hub
[    0.085000] usbcore: registered new device driver usb
[    0.090000] pps_core: LinuxPPS API ver. 1 registered
[    0.090000] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.090000] PTP clock support registered
[    0.090000] EDAC MC: Ver: 3.0.0
[    0.090000] clocksource: Switched to clocksource timer
[    0.090000] simple-framebuffer 5ff80000.framebuffer: framebuffer at 0x5ff80000, 0x7f800 bytes, mapped to 0xe0080000
[    0.090000] simple-framebuffer 5ff80000.framebuffer: format=x8r8g8b8, mode=480x272x32, linelength=1920
[    0.095000] Console: switching to colour frame buffer device 60x34
[    0.095000] simple-framebuffer 5ff80000.framebuffer: fb0: simplefb registered!
[    0.105000] NET: Registered protocol family 2
[    0.105000] TCP established hash table entries: 4096 (order: 2, 16384 bytes)
[    0.110000] TCP bind hash table entries: 4096 (order: 3, 32768 bytes)
[    0.115000] TCP: Hash tables configured (established 4096 bind 4096)
[    0.115000] UDP hash table entries: 256 (order: 1, 8192 bytes)
[    0.120000] UDP-Lite hash table entries: 256 (order: 1, 8192 bytes)
[    0.120000] NET: Registered protocol family 1
[    0.125000] RPC: Registered named UNIX socket transport module.
[    0.125000] RPC: Registered udp transport module.
[    0.130000] RPC: Registered tcp transport module.
[    0.130000] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    0.135000] Unpacking initramfs...
[    0.615000] Freeing initrd memory: 8440K (c97c2000 - ca000000)
[    0.625000] futex hash table entries: 256 (order: 2, 16384 bytes)
[    0.640000] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    0.645000] NFS: Registering the id_resolver key type
[    0.645000] Key type id_resolver registered
[    0.645000] Key type id_legacy registered
[    0.650000] ntfs: driver 2.1.32 [Flags: R/W].
[    0.650000] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 249)
[    0.655000] io scheduler noop registered
[    0.655000] io scheduler deadline registered
[    0.660000] io scheduler cfq registered (default)
[    0.660000] sun4i-usb-phy 1c13400.phy: could not find pctldev for node /soc@01c00000/pinctrl@01c20800/chip_id_det_pin@0, deferring probe
[    0.670000] sun5i-a13-pinctrl 1c20800.pinctrl: initialized sunXi PIO driver
[    0.680000] backlight supply power not found, using dummy regulator
[    0.695000] coupled-voltage-regulator wifi_reg: Couldn't get regulator vin0
[    0.755000] Serial: 8250/16550 driver, 4 ports, IRQ sharing disabled
[    0.780000] 1c28400.serial: ttyS0 at MMIO 0x1c28400 (irq = 28, base_baud = 1500000) is a U6_16550A
[    1.450000] console [ttyS0] enabled
[    1.475000] 1c28c00.serial: ttyS1 at MMIO 0x1c28c00 (irq = 29, base_baud = 1500000) is a U6_16550A
[    1.490000] SuperH (H)SCI(F) driver initialized
[    1.495000] msm_serial: driver initialized
[    1.500000] STMicroelectronics ASC driver initialized
[    1.510000] [drm] Initialized drm 1.1.0 20060810
[    1.520000] panel supply power not found, using dummy regulator
[    1.535000] loop: module loaded
[    1.550000] nand: device found, Manufacturer ID: 0x98, Chip ID: 0xd7
[    1.560000] nand: Toshiba TC58TEG5DCLTA00
[    1.565000] nand: 4096 MiB, MLC, erase size: 4096 KiB, page size: 16384, OOB size: 1280
[    1.580000] Scanning device for bad blocks
[    1.595000] Bad eraseblock 60 at 0x00000f000000
[    1.600000] Bad eraseblock 82 at 0x000014800000
[    1.610000] Bad eraseblock 86 at 0x000015800000
[    1.620000] Bad eraseblock 121 at 0x00001e400000
[    1.630000] Bad eraseblock 128 at 0x000020000000
[    1.665000] Bad eraseblock 297 at 0x00004a400000
[    1.670000] Bad eraseblock 300 at 0x00004b000000
[    1.685000] Bad eraseblock 362 at 0x00005a800000
[    1.705000] Bad eraseblock 416 at 0x000068000000
[    1.725000] Bad eraseblock 517 at 0x000081400000
[    1.740000] Bad eraseblock 548 at 0x000089000000
[    1.745000] random: nonblocking pool is initialized
[    1.770000] Bad eraseblock 653 at 0x0000a3400000
[    1.780000] Bad eraseblock 671 at 0x0000a7c00000
[    1.800000] Bad eraseblock 751 at 0x0000bbc00000
[    1.810000] Bad eraseblock 788 at 0x0000c5000000
[    1.825000] Bad eraseblock 822 at 0x0000cd800000
[    1.835000] Bad eraseblock 857 at 0x0000d6400000
[    1.855000] Bad eraseblock 932 at 0x0000e9000000
[    1.875000] Bad eraseblock 1012 at 0x0000fd000000
[    1.880000] Bad eraseblock 1015 at 0x0000fdc00000
[    1.885000] Bad eraseblock 1017 at 0x0000fe400000
[    1.890000] 5 ofpart partitions found on MTD device 1c03000.nand
[    1.900000] Creating 5 MTD partitions on "1c03000.nand":
[    1.905000] 0x000000000000-0x000000400000 : "SPL"
[    1.915000] 0x000000400000-0x000000800000 : "SPL.backup"
[    1.920000] 0x000000800000-0x000000c00000 : "U-Boot"
[    1.930000] 0x000000c00000-0x000001000000 : "env"
[    1.935000] 0x000001000000-0x000200000000 : "rootfs"
[    1.940000] mtd: partition "rootfs" extends beyond the end of device "1c03000.nand" -- size truncated to 0xff000000
[    1.955000] libphy: Fixed MDIO Bus: probed
[    1.965000] CAN device driver interface
[    1.970000] igb: Intel(R) Gigabit Ethernet Network Driver - version 5.3.0-k
[    1.980000] igb: Copyright (c) 2007-2014 Intel Corporation.
[    1.985000] pegasus: v0.9.3 (2013/04/25), Pegasus/Pegasus II USB Ethernet driver
[    1.995000] usbcore: registered new interface driver pegasus
[    2.005000] usbcore: registered new interface driver asix
[    2.010000] usbcore: registered new interface driver ax88179_178a
[    2.020000] usbcore: registered new interface driver cdc_ether
[    2.030000] usbcore: registered new interface driver smsc75xx
[    2.035000] usbcore: registered new interface driver smsc95xx
[    2.045000] usbcore: registered new interface driver net1080
[    2.055000] usbcore: registered new interface driver rndis_host
[    2.065000] usbcore: registered new interface driver cdc_subset
[    2.075000] usbcore: registered new interface driver zaurus
[    2.080000] usbcore: registered new interface driver cdc_ncm
[    2.090000] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    2.100000] ehci-pci: EHCI PCI platform driver
[    2.110000] ehci-platform: EHCI generic platform driver
[    2.115000] ehci-omap: OMAP-EHCI Host Controller driver
[    2.125000] ehci-orion: EHCI orion driver
[    2.130000] SPEAr-ehci: EHCI SPEAr driver
[    2.135000] ehci-st: EHCI STMicroelectronics driver
[    2.140000] ehci-exynos: EHCI EXYNOS driver
[    2.150000] ehci-atmel: EHCI Atmel driver
[    2.155000] tegra-ehci: Tegra EHCI driver
[    2.160000] ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
[    2.170000] ohci-pci: OHCI PCI platform driver
[    2.175000] ohci-platform: OHCI generic platform driver
[    2.185000] ohci-omap3: OHCI OMAP3 driver
[    2.190000] SPEAr-ohci: OHCI SPEAr driver
[    2.195000] ohci-st: OHCI STMicroelectronics driver
[    2.200000] ohci-atmel: OHCI Atmel driver
[    2.205000] usbcore: registered new interface driver usb-storage
[    2.215000] udc-core: couldn't find an available UDC - added [g_cdc] to list of pending drivers
[    2.230000] mousedev: PS/2 mouse device common for all mice
[    2.240000] input: 1c25000.rtp as /devices/platform/soc@01c00000/1c25000.rtp/input/input0
[    2.255000] i2c /dev entries driver
[    2.260000] axp20x 0-0034: AXP20x variant AXP209 found
[    2.275000] axp20x-gpio axp20x-gpio: AXP209 GPIO driver loaded
[    2.295000] input: axp20x-pek as /devices/platform/soc@01c00000/1c2ac00.i2c/i2c-0/0-0034/axp20x-pek/input/input1
[    2.315000] axp20x 0-0034: AXP20X driver loaded
[    2.325000] input: tca8418 as /devices/platform/soc@01c00000/1c2b000.i2c/i2c-1/1-0034/input/input2
[    2.340000] pcf857x 2-0038: probed
[    2.345000] Driver for 1-wire Dallas network protocol.
[    2.375000] sunxi-wdt 1c20c90.watchdog: Watchdog enabled (timeout=16 sec, nowayout=0)
[    2.385000] sdhci: Secure Digital Host Controller Interface driver
[    2.395000] sdhci: Copyright(c) Pierre Ossman
[    2.405000] Synopsys Designware Multimedia Card Interface Driver
[    2.415000] sdhci-pltfm: SDHCI platform and OF driver helper
[    2.430000] ledtrig-cpu: registered to indicate activity on CPUs
[    2.440000] hidraw: raw HID events driver (C) Jiri Kosina
[    2.445000] usbcore: registered new interface driver usbhid
[    2.455000] usbhid: USB HID core driver
[    2.465000] ip_tables: (C) 2000-2006 Netfilter Core Team
[    2.475000] NET: Registered protocol family 10
[    2.480000] sit: IPv6 over IPv4 tunneling driver
[    2.490000] NET: Registered protocol family 17
[    2.495000] can: controller area network core (rev 20120528 abi 9)
[    2.505000] NET: Registered protocol family 29
[    2.510000] can: raw protocol (rev 20120528)
[    2.515000] can: broadcast manager protocol (rev 20120528 t)
[    2.525000] can: netlink gateway (rev 20130117) max_hops=1
[    2.535000] Key type dns_resolver registered
[    2.545000] ThumbEE CPU extension supported.
[    2.550000] Registering SWP/SWPB emulation handler
[    2.560000] usb0-vbus: supplied by vcc5v0
[    2.570000] [drm] Supports vblank timestamp caching Rev 2 (21.10.2013).
[    2.580000] [drm] No driver support for vblank timestamp query.
[    2.590000] sun4i-drm display-engine: bound 1e60000.display-backend (ops 0xc10a3d54)
[    2.600000] sun4i-drm display-engine: bound 1c0c000.lcd-controller (ops 0xc10a3bf8)
[    2.610000] sun4i-drm display-engine: bound 1c0a000.tv-encoder (ops 0xc10a3e30)
[    2.620000] fb: switching to sun4i-drm-fb from simple
[    2.630000] Console: switching to colour dummy device 80x30
[    2.655000] Console: switching to colour frame buffer device 60x34
[    2.660000] sun4i-drm display-engine: fb0:  frame buffer device
[    2.670000] ehci-platform 1c14000.usb: EHCI Host Controller
[    2.680000] ehci-platform 1c14000.usb: new USB bus registered, assigned bus number 1
[    2.690000] ehci-platform 1c14000.usb: irq 22, io mem 0x01c14000
[    2.710000] ehci-platform 1c14000.usb: USB 2.0 started, EHCI 1.00
[    2.715000] hub 1-0:1.0: USB hub found
[    2.725000] hub 1-0:1.0: 1 port detected
[    2.730000] ohci-platform 1c14400.usb: Generic Platform OHCI controller
[    2.740000] ohci-platform 1c14400.usb: new USB bus registered, assigned bus number 2
[    2.750000] ohci-platform 1c14400.usb: irq 23, io mem 0x01c14400
[    2.820000] hub 2-0:1.0: USB hub found
[    2.825000] hub 2-0:1.0: 1 port detected
[    2.835000] usb_phy_generic.0.auto supply vcc not found, using dummy regulator
[    2.845000] musb-hdrc musb-hdrc.1.auto: MUSB HDRC host driver
[    2.855000] musb-hdrc musb-hdrc.1.auto: new USB bus registered, assigned bus number 3
[    2.865000] hub 3-0:1.0: USB hub found
[    2.870000] hub 3-0:1.0: 1 port detected
[    2.880000] using random self ethernet address
[    2.885000] using random host ethernet address
[    2.890000] using host ethernet address: f8:dc:7a:00:00:01
[    2.895000] using self ethernet address: f8:dc:7a:00:00:02[    2.905000] usb0: HOST MAC f8:dc:7a:00:00:01
[    2.915000] usb0: MAC f8:dc:7a:00:00:02
[    2.920000] g_cdc gadget: CDC Composite Gadget, version: King Kamehameha Day 2008
[    2.930000] g_cdc gadget: g_cdc ready
[    2.935000] sunxi-mmc 1c0f000.mmc: No vqmmc regulator found
[    2.945000] sunxi-mmc 1c0f000.mmc: allocated mmc-pwrseq
[    2.985000] sunxi-mmc 1c0f000.mmc: base:0xe014c000 irq:20
[    2.990000] hctosys: unable to open rtc device (rtc0)
[    3.000000] of_cfs_init
[    3.000000] of_cfs_init: OK
[    3.010000] vcc3v3: disabling
[    3.015000] usb0-vbus: disabling
[    3.025000] Freeing unused kernel memory: 1068K (c0f03000 - c100e000)
[    3.035000] sunxi-mmc 1c0f000.mmc: smc 0 err, cmd 8, RTO !!
Loading, please wait...
[    3.060000] mmc0: new high speed SDIO card at address 0001
[    3.075000] usb 1-1: new high-speed USB device number 2 using ehci-platform
[    3.150000] systemd-udevd[78]: starting version 215
[    3.220000] hub 1-1:1.0: USB hub found
[    3.230000] hub 1-1:1.0: 4 ports detected
[    3.360000] g_cdc gadget: high-speed config #1: CDC Composite (ECM + ACM)
```

### Mounting and switching to root filesystem at NFS

```
Begin: Loading essential drivers ... done.
Begin: Running /scripts/init-premount ... done.
Begin: Mounting root file system ... Begin: Running /scripts/nfs-top ... done.
Begin: Running /scripts/nfs-premount ... done.
IP-Config: usb0 hardware address f8:dc:7a:00:00:02 mtu 1500 DHCP RARP
IP-Config: usb0 guessed broadcast address 192.168.0.255
IP-Config: usb0 complete (dhcp from 192.168.0.1):
 address: 192.168.0.100    broadcast: 192.168.0.255    netmask: 255.255.255.0
 gateway: 192.168.0.1      dns0     : 192.168.1.254    dns1   : 8.8.8.8
 domain : attlocal.net
 rootserver: 192.168.0.1 rootpath: 192.168.0.1:/srv/pxe/rootfs
 filename  : pxelinux.0
done.
Begin: Running /scripts/nfs-bottom ... done.
Begin: Running /scripts/init-bottom ... done.
[    5.910000] systemd[1]: systemd 215 running in system mode. (+PAM +AUDIT +SELINUX +IMA +SYSVINIT +LIBCRYPTSETUP +GCRYPT +ACL +XZ -SECCOMP -APPARMOR)
[    5.930000] systemd[1]: Detected architecture 'arm'.

Welcome to Debian GNU/Linux 8 (jessie)!

[    5.945000] systemd[1]: Set hostname to <chip>.
[    6.470000] systemd[1]: Expecting device dev-ttyS0.device...
[    6.480000] systemd[1]: Expecting device dev-ttyGS0.device...

[    6.495000] systemd[1]: Starting Forward Password Requests to Wall Directory Watch.

[    6.495000] systemd[1]: Started Forward Password Requests to Wall Directory Watch.
[    6.495000] systemd[1]: Starting Remote File Systems (Pre).
[    6.525000] systemd[1]: Reached target Remote File Systems (Pre).
[    6.525000] systemd[1]: Starting Encrypted Volumes.
[    6.540000] systemd[1]: Reached target Encrypted Volumes.
[    6.540000] systemd[1]: Set up automount Arbitrary Executable File Formats File System Automount Point.
[    6.540000] systemd[1]: Starting Swap.
[    6.565000] systemd[1]: Reached target Swap.
[    6.575000] systemd[1]: Starting Root Slice.e).

[  OK  ] Reached target Encrypted Volumes.
[  [    6.585000] systemd[1]: Created slice Root Slice.
OK  ] Reached target Swap.
[    6.585000] systemd[1]: Starting /dev/initctl Compatibility Named Pipe.
[    6.605000] systemd[1]: Listening on /dev/initctl Compatibility Named Pipe.
[    6.605000] systemd[1]: Starting Delayed Shutdown Socket.
[    6.625000] systemd[1]: Listening on Delayed Shutdown Socket.
[    6.625000] systemd[1]: Starting Journal Socket (/dev/log).
[    6.645000] systemd[1]: Listening on Journal Socket (/dev/log).
[    6.645000] systemd[1]: Starting User and Session Slice.
[    6.660000] systemd[1]: Created slice User and Session Slice.
[    6.660000] systemd[1]: Starting udev Control Socket.
[    6.675000] systemd[1]: Listening on udev Control Socket.
[    6.675000] systemd[1]: Starting udev Kernel Socket.
[    6.690000] systemd[1]: Listening on udev Kernel Socket.
[    6.690000] systemd[1]: Starting Journal Socket.
[    6.705000] systemd[1]: Listening on Journal Socket.
[    6.705000] systemd[1]: Starting System Slice.
[    6.720000] systemd[1]: Created slice System Slice.
[    6.720000] systemd[1]: Starting Increase datagram queue length...
[  OK  ] Created slice Root Slice.
[  OK  ] Listening on /dev/initctl Compatibility Named Pipe.
[  OK  ] Listening on Delayed Shutdown Socket[    6.745000] systemd[1]: Starting Restore / save the current clock...
.
[  OK  ] Listening on Journal Socket (/dev/log).
[  OK  ] Created slice User and Session Slice.
[  OK  ] Listening on udev Control Socket.
[  OK  ] Listening on udev Kernel Socket.
[  OK  ] Listening on Journal Socket.
[  OK  ] Created slice System Slice.
         Starting Increase datagram queue length...
[    6.875000] systemd[1]: Started Set Up Additional Binary Formats.
[    6.875000] systemd[1]: Mounting Debug File System...
[    6.905000] systemd[1]: Mounted Huge Pages File System.
         Starting Restore / save the current clock...
         Mounting Debug File System...
[    6.975000] systemd[1]: Starting Load Kernel Modules...
[    7.020000] systemd[1]: Starting udev Coldplug all Devices...
[    7.045000] systemd[1]: Starting Create list of required static device nodes for the current kernel...
[    7.105000] systemd[1]: Mounting POSIX Message Queue File System...
         Starting Load Kernel Modules...
         Starting udev Coldplug all Devices...
         Starting Create list of required static device nodes...rrent kernel...
[    7.160000] systemd[1]: Starting system-getty.slice.
[    7.205000] systemd[1]: Created slice system-getty.slice.
[    7.205000] systemd[1]: Starting system-serial\x2dgetty.slice.
[    7.300000] systemd[1]: Created slice system-serial\x2dgetty.slice.
[    7.300000] systemd[1]: Starting Slices.
[    7.315000] fuse init (API version 7.23)
[    7.460000] Found ARM Mali 400 MP1 (r1p1)
[    7.460000] Allwinner sunXi mali glue initialized
[    7.525000] systemd[1]: Reached target Slices.
[    7.540000] systemd[1]: Mounted POSIX Message Queue File System.
[    7.555000] systemd[1]: Mounted Debug File System.
[    7.570000] systemd[1]: Started Increase datagram queue length.
         Mounting POSIX Message Queue File System...
[  OK  ] Created slice system-getty.slice.
[  OK  ] Created slice system-serial\x2dgetty.slice.
[  [    7.605000] systemd[1]: Started Restore / save the current clock.
OK  ] Reached target Slices.
[  OK  ] Mounted POSIX Message Queue File System.
[  OK  ] Mounted Debug File System.
[  OK  ] Started Increase datagram queue length.
[    7.635000] systemd[1]: Started Create list of required static device nodes for the current kernel.
[    7.635000] systemd[1]: Time has been changed
[  OK  ] Started Restore / save the current clock.
[  OK  ] Started Create list of required static device nodes ...current kernel.
[    7.735000] Mali: Mali device driver loaded
[    7.765000] systemd[1]: Started Load Kernel Modules.

[    7.895000] systemd[1]: Mounting Configuration File System...
         Mounting Configuration File System...
[    7.925000] systemd[1]: Starting Apply Kernel Variables...
         Starting Apply Kernel Variables...
[    7.940000] systemd[1]: Mounting FUSE Control File System...
         Mounting FUSE Control File System...
[    8.010000] systemd[1]: Starting Create Static Device Nodes in /dev...
         Starting Create Static Device Nodes in /dev...
[    8.030000] systemd[1]: Starting Syslog Socket.
[  OK  ] Listening on Syslog Socket.
[    8.090000] systemd[1]: Listening on Syslog Socket.
[    8.115000] systemd[1]: Starting Journal Service...
         Starting Journal Service...
[  OK  ] Started Journal Service.
[    8.155000] systemd[1]: Started Journal Service.
[  OK  ] Mounted FUSE Control File System.
[  OK  ] Mounted Configuration File System.
[  OK  ] Started udev Coldplug all Devices.
[  OK  ] Started Apply Kernel Variables.
[  OK  ] Started Create Static Device Nodes in /dev.
         Starting udev Kernel Device Manager...
[    8.500000] systemd-udevd[161]: starting version 215
[  OK  ] Started udev Kernel Device Manager.
         Starting Copy rules generated while the root was ro...
         Starting LSB: Set preliminary keymap...
[  OK  ] Started LSB: Set preliminary keymap.
         Starting Show Plymouth Boot Screen...
         Starting Remount Root and Kernel File Systems...
[  OK  ] Started Copy rules generated while the root was ro.
[  OK  ] Started Remount Root and Kernel File Systems.
[  OK  ] Started Show Plymouth Boot Screen.
[  OK  ] Reached target Paths.
         Starting Load/Save Random Seed...
[  OK  ] Reached target Local File Systems (Pre).
[  OK  ] Reached target Local File Systems.
         Starting Create Volatile Files and Directories...
         Starting Tell Plymouth To Write Out Runtime Data...
[  OK  ] Reached target Remote File Systems.
         Starting LSB: Set console font and keymap...
[  OK  ] Started Load/Save Random Seed.
[  OK  ] Started Tell Plymouth To Write Out Runtime Data.
[  OK  ] Started LSB: Set console font and keymap.
         Starting LSB: Raise network interfaces....
[  OK  ] Started Create Volatile Files and Directories.
         Starting Update UTMP about System Boot/Shutdown...
[  OK  ] Started Update UTMP about System Boot/Shutdown.
[  OK  ] Created slice system-systemd\x2dbacklight.slice.
         Starting Load/Save Screen Backlight Brightness of ba...ht:backlight...
[   11.525000] w1_master_driver w1_bus_master1: Family 2d for 2d.000012a47403.5e is not registered.
[  OK  ] Started Load/Save Screen Backlight Brightness of backlight:backlight.
[  OK  ] Found device /dev/ttyS0.
[   12.470000] sun4i-codec 1c22c00.codec: Codec <-> 1c22c00.codec mapping ok
[  OK  ] Created slice system-bt_rtk_hciattach.slice.
         Starting Tell Plymouth To Write Out Runtime Data...
         Starting Show Plymouth Boot Screen...
[  OK  ] Started Tell Plymouth To Write Out Runtime Data.
[  OK  ] Started Show Plymouth Boot Screen.
[   15.245000] cfg80211: World regulatory domain updated:
[   15.255000] cfg80211:  DFS Master region: unset
[   15.260000] cfg80211:   (start_freq - end_freq @ bandwidth), (max_antenna_gain, max_eirp), (dfs_cac_time)
[   15.270000] cfg80211:   (2402000 KHz - 2472000 KHz @ 40000 KHz), (N/A, 2000 mBm), (N/A)
[   15.280000] cfg80211:   (2457000 KHz - 2482000 KHz @ 40000 KHz), (N/A, 2000 mBm), (N/A)
[   15.295000] cfg80211:   (2474000 KHz - 2494000 KHz @ 20000 KHz), (N/A, 2000 mBm), (N/A)
[   15.305000] cfg80211:   (5170000 KHz - 5250000 KHz @ 80000 KHz, 160000 KHz AUTO), (N/A, 2000 mBm), (N/A)
[   15.315000] cfg80211:   (5250000 KHz - 5330000 KHz @ 80000 KHz, 160000 KHz AUTO), (N/A, 2000 mBm), (0 s)
[   15.330000] cfg80211:   (5490000 KHz - 5730000 KHz @ 160000 KHz), (N/A, 2000 mBm), (0 s)
[   15.340000] cfg80211:   (5735000 KHz - 5835000 KHz @ 80000 KHz), (N/A, 2000 mBm), (N/A)
[   15.350000] cfg80211:   (57240000 KHz - 63720000 KHz @ 2160000 KHz), (N/A, 0 mBm), (N/A)
[  OK  ] Found device /dev/ttyGS0.
[  OK  ] Started LSB: Raise network interfaces..
[  OK  ] Created slice system-systemd\x2drfkill.slice.
         Starting Load/Save RF Kill Switch Status of rfkill1...
         Starting Load/Save RF Kill Switch Status of rfkill0...
[  OK  ] Reached target Sound Card.
[  OK  ] Reached target Network.
[  OK  ] Reached target Network is Online.
[  OK  ] Started Load/Save RF Kill Switch Status of rfkill1.
[  OK  ] Started Load/Save RF Kill Switch Status of rfkill0.
[  OK  ] Reached target System Initialization.
[  OK  ] Listening on Avahi mDNS/DNS-SD Stack Activation Socket.
[  OK  ] Listening on D-Bus System Message Bus Socket.
[  OK  ] Reached target Sockets.
[  OK  ] Reached target Timers.
         Starting Restore Sound Card State...
[  OK  ] Reached target Basic System.
         Starting 8723bs Bluetooth hciattach...
         Starting NAND health watchdog service...
[  OK  ] Started NAND health watchdog service.
         Starting Network Manager...
         Starting Regular background program processing daemon...
[  OK  ] Started Regular background program processing daemon.
         Starting Restore /etc/resolv.conf if the system cras...s shut down....
         Starting System Logging Service...
         Starting Avahi mDNS/DNS-SD Stack...
         Starting /etc/rc.local Compatibility...
         Starting Permit User Sessions...
         Starting D-Bus System Message Bus...
[  OK  ] Started D-Bus System Message Bus.
[  OK  ] Started Avahi mDNS/DNS-SD Stack.
         Starting Login Service...
         Starting LSB: Start NTP daemon...
[  OK  ] Started System Logging Service.
[  OK  ] Started Restore Sound Card State.
[  OK  ] Started 8723bs Bluetooth hciattach.
[  OK  ] Started Restore /etc/resolv.conf if the system crash...was shut down..
[  OK  ] Started Permit User Sessions.
[  OK  ] Started Login Service.
[   20.755000] Bluetooth: Core ver 2.21
[   20.755000] Bluetooth: Starting self testing
[   20.815000] Bluetooth: ECDH test passed in 54884 usecs
         Starting Authenticate and Authorize Users to Run Privileged Tasks...
         Stopping NAND health watchdog service...
[  OK  ] Stopped NAND health watchdog service.
         Starting NAND health watchdog service...
[  OK  ] Started NAND health watchdog service.
         Starting battery polling for pocket-wm...
         Starting Light Display Manager...
         Stopping NAND health watchdog service...
[  OK  ] Stopped NAND health watchdog service.
         Starting NAND health watchdog service...
[  OK  ] Started NAND health watchdog service.
[   21.795000] Bluetooth: SMP test passed in 149 usecs
[   21.800000] Bluetooth: Finished self testing
[   21.805000] NET: Registered protocol family 31
[   21.810000] Bluetooth: HCI device and connection manager initialized
[   21.820000] Bluetooth: HCI socket layer initialized
[   21.825000] Bluetooth: L2CAP socket layer initialized
[   21.830000] Bluetooth: SCO socket layer initialized
         Stopping NAND health watchdog service...
[  OK  ] Stopped NAND health watchdog service.
         Starting NAND health watchdog service...
[  OK  ] Started NAND health watchdog service.
[  OK  ] Started Authenticate and Authorize Users to Run Privileged Tasks.
[   22.170000] Bluetooth: HCI UART driver ver 2.3
[   22.175000] Bluetooth: HCI UART protocol H4 registered
[   22.185000] Bluetooth: HCI UART protocol BCSP registered
[   22.190000] Bluetooth: HCI UART protocol LL registered
[   22.195000] Bluetooth: HCI UART protocol ATH3K registered
[   22.200000] Bluetooth: HCI UART protocol Three-wire (H5) registered
[   22.210000] Bluetooth: HCI UART protocol Intel registered
[   22.220000] Bluetooth: HCI UART protocol BCM registered
         Stopping NAND health watchdog service...
[  OK  ] Stopped NAND health watchdog service.
         Starting NAND health watchdog service...
[  OK  ] Started NAND health watchdog service.
         Stopping NAND health watchdog service...
[  OK  ] Stopped NAND health watchdog service.
         Starting NAND health watchdog service...
[FAILED] Failed to start NAND health watchdog service.
See 'systemctl status ubihealthd.service' for details.
[   23.970000] Bluetooth: BNEP (Ethernet Emulation) ver 1.3
[   23.980000] Bluetooth: BNEP socket layer initialized

Debian GNU/Linux 8 chip ttyS0

```

### Boot finishes booting from NFS via USB Ethernet networking

```
chip login: chip
Password:
```
```
Last login: Thu Jan  1 04:16:51 UTC 1970 on ttyS0
Linux chip 4.4.13-ntc-mlc #1 SMP Tue Dec 6 21:38:00 UTC 2016 armv7l

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
```
```
chip@chip:~$ df .
```
```
Filesystem                       1K-blocks     Used Available Use% Mounted on
192.168.0.1:/srv/nfs/rootfs/b126 122700544 60582400  55869440  53% /
```
```
chip@chip:~$ ip a | grep -A1 usb
```
```
3: usb0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether f8:dc:7a:00:00:02 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.100/24 brd 192.168.0.255 scope global usb0
       valid_lft forever preferred_lft forever

```

# REFERENCES:

- [Use U-Boot PXE Boot and extlinux.conf](https://docs.u-boot.org/en/latest/usage/pxe.html)
- [How to configure a Raspberry Pi as a PXE boot server](https://linuxconfig.org/how-to-configure-a-raspberry-pi-as-a-pxe-boot-server)
- [Booting a new kernel and rootfs via NFS](https://github.com/CHIPFriends/CHIP-Wiki/wiki/Booting-a-new-kernel-and-rootfs-via-NFS)


