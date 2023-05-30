# Running Pi-Star on the Libre Computer AML-S905X "Le Potato"

The Libre Computer Le Potato is a SBC comparable to a RPi3. It is not directly Pi compatible, but does have a fair amount of compatibility with software. It's GPIO header
shares a number of pins with the layout of the header on the Pi. Libre provides some scripts to help "rewire" GPIO pins as well as make Rasbian based images boot on their
hardware. I believe the only major thing it's doing is adding it's bootloader to the boot sector, installing grub-efi, and updating the kernel to something not Pi-Specific. Unlike the RPi; this SBC seems to boot using EFI and has an actual bootscreen/BIOS. 

You will need an RPi to run the conversion script. You will also need a linux distro that can mount and modify the partitions on your SDCard. 

The basic steps are this:

- Flash a new copy of Pi-Star to a sdcard
- Create a new 300MB FAT32 partition at the end of the SDCard
- Copy the files from the original /boot partition to the new partition.
- Set the new /boot partition as bootable
- Modify Pi-Stars /etc/fstab to handle device name changes and new /boot
- Boot Pi-Star on an RPi
- Run Libre's conversion script
- Boot Pi-Star on Le Potato
- Install their gpio rewiring tool
- Enable UART on the GPIO header
- Configure Pi-Star/MMDVMHost for the correct serial port
- Configure your hotspot as usual

You may or may not maintain the ability to flash the hotspot's frimware. This uses pins 38 and 40 on the RPi header to reset the MMDVM and enable it's boot0 mode. Due to
difficult to diagnose issues during my setup; I thought those equivalent pins on the Le Potato may have been tied to lines I didn't have control over. I cut them off the header. If these pins can in fact be rewired by Libre's scripts and function, let me know. nq4t(at)nq4t.com

As a side effect, this will also update your kernel to 6.1.something. This doesn't seem to affect Pi-Star.

## New /boot partition

The reason we flash a new copy of Pi-Star is to make life easier creating a new partition. While the partition is only about 1.7 gigs when flashed, this is expanded to fill
card on first boot. You can probably move the data around to shrink the volume; but meh. It's easiest to just modify the partition using a seperate Linux system before the first boot. I put it at 300MB because I didn't know how much space I'd actually need. You'll also need to set this new partition as the boot partition.

Format it as FAT32 to match the original and copy the data over. I literally did it this way:

```
cp -R /mnt/p1/* /mnt/p3
```

## Modify Pi-Star's /etc/fstab

Not only do we need to tell the OS that our /boot mountpoint has changed, but the dev names for the sdcard are different. On the Pi, it's /dev/mmcblk0. On the Potato, it's
/dev/mmcblk1; since it also has an mmc port. Since these are specified in fstab as dev names and not UUID; the whole thing will fail to boot. The *best* solution is to just
switch /etc/fstab to use the UUID.

First, get the UUID for your partitions:

```
pi-star@pi-star:~$ sudo blkid
/dev/mmcblk0: PTUUID="a0e29a9d" PTTYPE="dos"
/dev/mmcblk0p1: SEC_TYPE="msdos" LABEL_FATBOOT="boot" LABEL="boot" UUID="22E0-C711" TYPE="vfat" PARTUUID="a0e29a9d-01"
/dev/mmcblk0p2: LABEL="pi-star" UUID="f5cd5220-f083-4b00-b988-30075b8f3904" TYPE="ext4" PARTUUID="a0e29a9d-02"
/dev/mmcblk0p3: SEC_TYPE="msdos" UUID="5FAC-A287" TYPE="vfat" PARTUUID="a0e29a9d-03"
```

Now mount the second/ext4 partition to your system and load /etc/fstab in an editor. Modify the mount-points for / and /boot:

```
UUID=5FAC-A287          /boot                   vfat    defaults,ro                                     0       2
UUID=f5cd5220-f083-4b00-b988-30075b8f3904       /                       ext4    defaults,noatime,ro                             0       1
```

Substiting, of course, the UUID for your partitions. Save the file and unmount all the partitions. 

## Run Script To Modify OS

Stick the card in your RPi and boot it up. After it boots, ssh in to it; don't touch anything on the browser dashboard. We need to git the repository with the script and
run it. 

```
git clone https://github.com/libre-computer-project/libretech-raspbian-portability.git lrp
sudo lrp/oneshot.sh aml-s905x-cc
```

The script will do a number of things and hopefully report no errors, asking you to press any key to shut down. After your Pi is shut down, put the card in Le Potato. If
all goes well, you should get the green LED light up a few seconds later; or just watch the bootscreen's output with a monitor. 

## Enable UART-A

The UART on the GPIO header is not enabled by default. So we need to install the wiring tool to fix that.

```
git clone https://github.com/libre-computer-project/libretech-wiring-tool.git
cd libretech-wiring-tool/
sudo ./install.sh
sudo ldto enable uart-a
```

## Configure Pi-Star For Serial Port

The UART on the GPIO is located at /dev/ttyAML6. You can supposedly set this on the configuration menu by picking it from the "Port" dropdown next to Display Type:; which
makes it look more like it's for setting the display port and not the mmdvm port. The logs kept telling me it was still using the wrong port; so I set that to "modem" and
manually configured it under the "Modem" section under Expert -> MMDVMHost. 

If you're watching the live logs, you'll be able to see when it actually starts talking. Otherwise, just look for the firmware being displayed on the dashboard. 

Configure the rest of your hotspot.
