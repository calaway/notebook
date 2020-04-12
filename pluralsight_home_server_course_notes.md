# PluralSight Raspberry Pi Home Server Course Notes

Notes from the PluralSight Raspberry Pi Home Server Course

## Getting Started

### Choosing an OS

Raspbian is the officially supported Linux distro. Recommend the full (desktop) version for a home server.

Download and write the image according the the instructions on [RaspberryPi.org/downloads/raspbian](https://www.raspberrypi.org/downloads/raspbian/).

Instructions as of the time of the course:

On Windows you can use Win32DiskImager.

On Mac/Linux:
```bash
dd bs=4m if=xxx.img of=/dev/yyy
```

### SD Card Contents

Most files on the written image should not be touched, except for `/boot/config.txt` which contains editable configuration settings.

### Elevated Privileges

Update all sources:
```bash
sudo apt-get update
```

Upgrade all software packages:
```bash
sudo apt-get upgrade
```

### Backing Up

On Windows you can again us Win32DiskImager as we did on install.

On Mac/Linux it is similar to the install command, but in reverse.
```bash
sudo dd if=/dev/diskn of=~/pi.img bs=1m
```

This is a bit-for-bit copy, which means a lot of it is wasted space. Compress it to save hard drive space.

Shutdown command:
```bash
sudo shutdown -h now # -h for "hard"
```

## Going Headless

### Introduction

In this module we will:
- Assign the Pi a static IP address on the home network
- Connect remotely via command line
- Connect remotely via the web
- Connect remotely via remote desktop

### Static IP Addresses

Your home network router uses DHCP (Dynamic Host Configuration Protocol) to assign IP addresses to devices on the network.

### Static IP Method 1

First, disable network interface names, if enabled.
1. `sudo raspi-config`
1. Network Options
1. Network Interface Names
1. Disable
1. Reboot to save any changes

We'll need to gather some information before proceeding.

Find your IP address with `ifconfig` (use `sudo` if necessary). Look under `eth0` or `wlan0` for ethernet or WiFi respectively. Alternatively, try `hostname -I`.

Find the address of your router with `sudo route -n` on the line with `Flags=UG` under the `Gateway` column.

Finde the list of DNS servers the Pi is using with `cat /etc/resolv.conf`.

Edit the DHCP config file with `sudo nano /etc/dhcpcd.conf`. Mimic the commented out "Example static IP configuation" section. Add this to the end of the file:
```
# `wlan0` or `eth0` for WiFi or ethernet, respectively
interface wlan
# The static IP you would like assigned
static ip_address=192.168.0.2/24
# The address of your router
static routers=192.168.0.1
# The list of DNS servers, space delimited
static domain_name_servers=8.8.8.8 8.8.4.4
```

Reboot for the changes to take effect. Find your IP address and DNS servers as above to verify the changes.

### Static IP Method 2

There are two methods to do this:
1) Via the DHCP configuration on the Pi. Instead of requesting an IP from the router you tell the router which IP to assign. Tutorial [here](https://pimylifeup.com/raspberry-pi-static-ip-address/)
2) Via the router's settings. Assign a static IP address to the Pi's MAC address.

Option 2 was recommended by the [Raspberry Pi Home Server PluralSight course](https://app.pluralsight.com/library/courses/raspberry-pi-home-server/table-of-contents), because all your static IP configurations across all devices are in one place, and you don't run the risk of a collision if you request an address that the router already assigned elsewhere.

Find the MAC address with `ifconfig` (may require `sudo`), under `wlan0` or `eth0` as the case may be.

On my CenturyLink Xyzel C3000Z router this can be set under Advanced Setup >> DHCP Reservation ([link](http://192.168.0.1/advancedsetup_dhcpreservation.html)).

### SSH

Use SSH (secure shell) to connect to the Raspberry Pi command line remotely.
- `sudo raspi-config`
- Interfacing Options
- SSH
- Enable

Connect from another computer:
```bash
ssh pi@192.168.0.2
```

### Webmin

Webmin is a web UI alternative to ssh to accomplish common administration tasks.

> [Webmin](http://www.webmin.com) is a web-based interface for system administration for Unix. Using any modern web browser, you can setup user accounts, Apache, DNS, file sharing and much more. Webmin removes the need to manually edit Unix configuration files like /etc/passwd, and lets you manage a system from the console or remotely.

Follow the installation instructions [here](http://www.webmin.com/deb.html) under the Using the Webmin APT repository section. The one change I made to their instructions is instead of adding the `sources` to `/etc/apt/sources.list`, create a new file to keep the sources more organized titled `/etc/apt/sources.list.d/webmin.list` and add it there.

Note: the Chrome & Brave browsers would not allow me to trust the `https` site without a certificate and open it anyway, but Firefox would.

### VNC Remote Desktop

We can connect to the Pi's desktop remotely using VNC (virtual network computing) via the RealVNC server that ships with Raspbian.

Enable VNC on the Pi:
- `sudo raspi-config` >>
- Interfacing Options >>
- VNC >>
- Enable

Download the RealVNC client from [their website](https://www.realvnc.com/en/) and follow the Getting Started instructions.

## Adding External Storage

### Partitioning the Drive

If necessary to power the external hard drive, the USB power output can be increased by editing the boot config with
```bash
sudo nano /boot/config.txt
```
and adding the following line to the end
```
max_usb_current=1
```

We can use `parted` to partition the external drive.

```bash
$ sudo parted
# Show all drives & partitions
(parted) print all
# Select the drive you want to partition
(parted) select /dev/sda
# Show just the selected drive
(parted) print
# Make a new partition table (guid partition table)
(parted) mklabel gpt
# Verify the new partition table exitsts
(parted) print
# Make a new partition that we can boot from, instead of from the SD card
(parted) mkpart
Partition name? []? os # Name the partition whatever you like
File system type? [ext2]? ext4
Start? 0gb
End? 16 gb
# Make a second and third partition using the single line syntax
(parted) mkpart data ext4 16gb 50%
(parted) mkpart data-ntfs ntfs 50% 100%
# Verify everything looks correct
(parted) print
# Quit parted
(parted) q
```

### Formatting the Data Partitions

After partioning the drive, we still need to format the partitions.
```bash
sudo mkfs.ext4 /dev/sda2
```

Install NTFS utilities.
```bash
sudo apt-get install ntfs-3g ntfs-config gdisk
```

Format the NTFS partition.
```bash
sudo mkfs.ntfs -Q -L data /dev/sda3
```

### Mounting the Drives

```bash
# Navigate to the directory where we wish to mound the partitions
cd /mnt
# Create a directory for each partition to act as a mount point
sudo mkdir data
sudo mkdir data-ntfs
# Add placeholder files to help verify it worked
sudo touch data/placeholder data-ntfs/placeholder
# Mount the partitions
sudo mount /dev/sda2 /mnt/data
sudo mount /dev/sda3 /mnt/data-ntfs
# Verify it worked by confirming we do _not_ see the placeholder files
ls data
ls data-ntfs
# Since this will not persist after a reboot, we can add these mounting instructions to the file system table (fstab) config
# Show the current contents of fstab
cat /etc/fstab
# Get the UUIDs for the partitions
sudo blkid
# Add the partition mount instructions to fstab
sudo nano /etc/fstab
```

The following three syntax options are equivalent.
```
/dev/disk/by-uuid/<<uuid>>  /mnt/data ext4  defaults/noatime  0  0
UUID=<<uuid>>               /mnt/data ext4  defaults/noatime  0  0
PARTUUID=<<partuuid>>       /mnt/data ext4  defaults/noatime  0  0
```

Reboot and confirm that the partitions are mounted by confirming that our placeholder files are not visible.

## Sharing Files with Samba

### Installing Samba

Samba is a way to share files using the SMB (server message block) protocol. CIFS or Common Internet File System is an implementation of the SMB protocol, but the two terms tend to be used interchangeably.

Installation instructions can be found at https://www.raspberrypi.org/documentation/remote-access/samba.md.

There are two packages needed that are both available from the package manager.
```bash
sudo apt-get install samba samba-common-bin
```

Make a public directory to share from. Then grant `write` permission to `group` and `other` to give access to everyone on the network.
```bash
sudo mkdir /mnt/data/public
```

### Linux Permissions

Files and directories have three sets of permissions that can be allowed or disallowed: read, write, and execute. There are three types of users that permissions can be assigned to: user, group, and other, which is everyone else.

Octal permissions make it possible to specify the permissions with a single number.
- Read = 4
- Write = 2 
- Execute = 1
Add these up to get the permissions for a user, group, or other. For example, octal permissions of `755` would give the user read, write, and execute; the group read and execute; and other read only permission. This is can also be written as `rwxr-xr--`.

A handy website to encode or decode octal permissions: http://permissions-calculator.org/

### Setting up Shares

To set up Samba shares on webmin:
1. Navigate to webmin at https://192.168.0.2:10000.
1. Since Samba was not installed when we originally setup webmin, it will be hidden away under Un-used Modules. Click Refresh Modules to get it to recognize Samba.
1. Navigate to the Samba config under Servers >> Samba Windows File Sharing.
1. Click on Windows Networking.
1. Ensure `Workgroup` is set to `WORKGROUP`. (Change this if your local Windows system has a different workgroup name than this default.)
1. Set `WINS mode` to `Be WINS server`. This stands for "Windows Internet Name Service," and it allows you to refer to other devices on the network by name instead of by address.
1. Click `Save` to save and return to the main Samba config page.
1. Back on the main Samba config page, click Create a New File Share.
1. Fill in a `Share name`, such as `public`.
1. Select the `public` directory created above under `Directory to share`.
1. Leave the other default settings. Click `Create` to save and return to the main Samba config page.
1. Select the new `public` share under the `Share Name` list.
1. Click `Security and Access Control`.
1. Set both `Writable?` and `Guest Access?` to `Yes`.
1. Click `Save` twice to return to the file share list.
1. Verify the new `public` has a `Security` setting of `Read/write to everyone`.
1. Click `Restart Samba Servers` for the settings to take effect across the network.

Note, I had to add `force user = pi` to the config in order to connect to a share on the external drive. The final config:
```
[public]
  writeable = yes
  path = /media/pi/calaway-2tb/public
  public = yes
  force user = pi
```

