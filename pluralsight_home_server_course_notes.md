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

__Note:__ I found it useful to _not_ skip this section, even though Raspbian mounted the drive autimatically, so you can specify the mount options in the `fstab` file.

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
/dev/disk/by-uuid/<<uuid>>  /mnt/data ext4  defaults,noatime  0  0
UUID=<<uuid>>               /mnt/data ext4  defaults,noatime  0  0
PARTUUID=<<partuuid>>       /mnt/data ext4  defaults,noatime  0  0
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

### Troubleshooting

Make sure directories are readable and executable for all users, including those leading up to the mount point. From [this article](https://pimylifeup.com/raspberry-pi-plex-server/):
```bash
sudo find /media/pi -type d -exec chmod go+rx {} \;
# A similar command can optionally be run for all files, but without making them executable.
sudo find /media/pi -type f -exec chmod go+r {} \;
```

If you're still having permissions issues after that you can add `force user = pi` to the config to connect as the `pi` user. The final config:
```
[public]
  writeable = yes
  path = /media/pi/calaway-2tb/public
  public = yes
  force user = pi
```

## Sharing Media with DLNA

### What is DLNA?

DLNA, the Digital Living Network Alliance is a standards body that developed a standard network protocol for sharing media.

### Choosing a Media Server

There are multiple DLNA implementations. We will discuss the most popular two for Raspberry Pi here, MiniDLNA and Plex.

__MiniDLNA, A.K.A ReadyMedia__
- Works on all Raspberry Pi models
- No user interface
- Webmin module available (but not very helpful)
- Not always up to date on `apt-get`
- Works better with some media players (like Windows Media Player)

__Plex Media Server__
- Only works on Pi 2 and above
- Fancier, full-featured interface
- Can transcode some (not all) media
- Needs some tweaks to set up
- Doesn't get along with Windows Media Player

Note that you cannot run both at the same time, since they serve on the same port.

### Installing MiniDLNA

Steps:
1. Install MiniDLNA via `apt-get`
1. Setup directories on the hard drive to house your media
1. Configure DLNA to say what kind of media is in each directory
1. Stream media throughout the house

Install:
```bash
sudo apt-get install minidlna
```
Note that the versions on Raspbian have not historically been kept up to date, so you might want to check your version and look into upgrading it before proceeding.

### Creating a Media Library

Create a directory for MiniDLNA to keep its database and logs. Then change the `owner` and `group` to `minidlna`. Let's put it at the root of the external drive.
```bash
# Navigate to the external hard drive.
cd /media/pi/{{external-hdd}}

# Make a new directory.
sudo mkdir minidlna

# Change the `owner` and `group` to `minidlna`
sudo chown minidlna:minidlna minidlna

# Make a new directory for music files in the shared `public` directory
cd public
mkdir music

# Change the `owner` and `group` to `nobody` and `nogroup`. Of course these aren't real users, but this will effectively assign ownership to everyone.
sudo chown nobody:nogroup music
```
These could also be created on another computer connecting to the Pi via Samba.

### Configuring MiniDLNA

Open the MiniDLNA config file for editing.
```bash
sudo nano /etc/minidlna.conf
```
Follow the templates in the file and add all media directories. It might look something like this.
```conf
media_dir=A, /media/pi/{{external-hdd1}}/public/music
media_dir=P, /media/pi/{{external-hdd1}}/public/pictures
media_dir=V, /media/pi/{{external-hdd1}}/public/video
media_dir=A, /media/pi/{{external-hdd2}}/public/music
media_dir=P, /media/pi/{{external-hdd2}}/public/pictures
media_dir=V, /media/pi/{{external-hdd2}}/public/video

# Set the database and log directories
db_dir=/media/pi/{{external-hdd}}/minidlna
log_dir=/media/pi/{{external-hdd}}/log
```

Restart the `minidlna` service to pick up the new configuration.
```bash
sudo service minidlna restart
# Check the status to see that it's running
sudo service minidlna status
# Check the logs
sudo cat /media/pi/{{external-hdd}}/minidlna/log/minidlna.log | more
# Note the warning the Inotify max_user_watches [8192] is low. Open the conf file to increase that.
sudo nano /etc/sysctl.conf
```

Add the following to the bottom of the file.
```conf
# minidlna tweaks
fs.inotify.max_user_watches = 65536 # Apparently the lowest number MiniDLNA won't complain about
```

Have MiniDLNA startup automatically whenever the server starts up.
```bash
sudo update-rc.d minidlna defaults
```

You can see what effect this is having on the Pi's processor(s) with `top` or `htop`.

### Updating MiniDLNA

Note that this is not necessary if the version in the apt-get sources is already up to date.

Update the sources file
```bash
sudo nano /etc/apt/sources.list
```

Copy the top line and change the version name to the current version.
```
deb http://raspbian.raspberrypi.org/raspbian/ wheezy main contrib non-free rpi
deb http://raspbian.raspberrypi.org/raspbian/ jessie main contrib non-free rpi
```
```bash
# Update all packages
sudo apt-get update

# Stop the minidlna service
sudo service minidlna stop

# Update minidlna with the same command we used to install it the first time
sudo apt-get install minidlna
```
This may ask to restart some services, which is fine, and may need to overwrite the config file. If so, make the config changes to `/etc/minidlna.conf` above again.

Then go back to `/etc/apt/sources.list` and comment out the `jessie` line and `sudo apt-get update` again to forget about the Jessie repository.

Check the installed version as well as the current remote version with `apt-cache policy minidlna`.

Reboot the server and make sure MiniDLNA is running as expected, `sudo reboot`.

### Plex Media Server

Plex is a little fancier and user friendls than MiniDLNA. It requires at least a Raspberry Pi 2 and provides a convenient web framework for configuration. It can also transcode media to play files that might otherwise be unsupported by the device playing them, but your mileage may vary on a low powered device like the Pi.

### Installing Plex Media Server

The basic steps are the same as for setting up MiniDLNA
1. Install
1. Setup directories
1. Configure
1. Stream media throughout the house

First of all, do not run both MiniDNLA and Plex at the same time because they'll fight over the network port.

It looks like the instructions on the video are outdated, so I'm going to instead follow the [instrucitons from Plex](https://support.plex.tv/articles/235974187-enable-repository-updating-for-supported-linux-server-distributions/). Instructions from the course are still outlined below, you know, for posterity.

Installation will look similar to `webmin`, where we needed to add and validate the source repository before installation.

```bash
# Change users to the super user
sudo su

# Download the signing key for the sources repository
cd /root
wget http://dev2day.de/pms/dev2day-pms.gpg.key

# Install the signing key
apt-key add dev2day-pms.gpg.key

# Exit super user
exit

# Create and edit a new sources file
sudo nano /etc/apt/sources.list.d/pms.list
```

Add the following line, matching the Debian version from `/etc/apt/sources.list`.
```
deb http://dev2day.de/pms/ wheezy main
```

```bash
# Update and install
sudo apt-get update
sudo apt-get install plexmediaserver

# Verify the service is running
sudo service plexmediaserver status
```

Follow the instructions from [Creating a Media Library](Creating-a-Media-Library) above to create directories for your media.

### Configuring Plex

Visit the Plex web UI at http://192.168.0.2:32400/web/.

If you choose to move the Plex database from the default location in the `/var/lib/plexmediaserver` directory onto the external drive, here are the instructions. (Note, I decided to leave it in the default location.)
```bash
# Stop the Plex Media Server service
sudo service plexmediaserver stop
# Check to see if there are any leftover processes owned by the plex user
htop
# Kill any leftover processes owned by the plex user
sudo killall -u plex
# Verify that they are all gone
htop
# Move the `plexmediaserver` directory
sudo mv /var/lib/plexmediaserver/ /mnt/data/plexmediaserver/ # Note that the trailing slashes can be important to Linux
# Verify that the files were moved successfully
ls -l /mnt/data/plexmediaserver
# Create a symbolic link from the old directory to the new directory
sudo ln -s /mnt/data/plexmediaserver/ /var/lib/plexmediaserver/
# Restart the Plex server
sudo service plexmediaserver restart
# You verify if Plex has any processes running
htop
```

Now you're ready to start adding your media directories as libraries in the Plex web UI. If you can't see the directories on the external drive, see troubleshooting under the Sharing Files with Samba above.

## Backing up Other Computers

Notes:
- An onsight backup is less safe than one offsite, since a disaster at home could take out both.
- Consider that you may prefer an encrypted backup.

Prerequisites:
- Hard drive
- Samba
- You may need to map a port through your firewall if your backup is offsite

### Windows Backup

```bash
# Create a directory for the backup
cd /mnt/data
sudo mkdir -p backups/desktop
# Give group and others write permissions
sudo chmod go+w backups backups/desktop
# Create a Samba share
sudo nano /etc/samba/smb.conf
```

Add a new share to the bottom of the file, giving it whatever name you prefer and copying the rest of the settings from the other shares we set up (remember to change the directory).
```conf
[backups]
        path = /mnt/data/backups/desktop
        writeable = yes
        public = yes
```

In Windows:
1. Open the backup options by hitting the Windows key and type `backup`
1. Click More Options >> See Advanced Options >> Select Drive >> Add Network Location >> browse to the location of the share >> Turn On
1. Back out to the backup settings and verify that automatic backup is enabled

Note that this does not currently support encryption.

### Duplicati

[Duplicati](https://www.duplicati.com/) is an open source backup option that requires no installation on the Pi. All the work is done by the computer being backed up. It can keep multiple versions and supports encryption.

Follow the instructions on their website to install Duplicati on the machine being backed up. Then open the web UI on localhost and follow the instructions to set up the backup.

It is also possible For Duplicati to manage the backup via ssh, rather than Samba.

### Remote Duplicati

If you don't mind opening a port on your firewall, you can communicate outsid of your local network. There are risks involved, so proceed at your own risk.

You could provide a strong password, but it is recommended to require key based authentication, which is far more secure, albeit less convenient.

The process is to generate a key pair on the client, and then install the public key on the server. Note the order, since it may be counterintuitive. This is about getting the server to trust the client, and not the other way around.

To generate a key pair, I recommend following the [instructions on GitHub](https://help.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh).

Copy the newly generated public key, then open the authorized keys file on the Pi with `sudo nano ~/.ssh/authorized_keys`, and paste the public key at the bottom of the file, underneath any other keys that were already there.

To stop accepting a password and require key based authentication, open the SSH config file with `sudo nano /etc/ssh/sshd_config`.
- Change `#PermitRootLogin prohibit-password` to `PermitRootLogin no`
- Uncomment `PubkeyAuthentication yes`
- (Verify the key works before you) Change `#PasswordAuthentication yes` to `PasswordAuthentication no`
- `ChallengeResponseAuthentication` should be set to `no`
- Change `UsePAM yes` to `UsePAM no`

Reload the SSH configuration with `sudo service ssh reload`.










