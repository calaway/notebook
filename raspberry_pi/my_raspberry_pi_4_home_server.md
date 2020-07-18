## Install Ubuntu Server

I installed Ubuntu 20.04 LTS (64-bit) from [here](https://ubuntu.com/download/raspberry-pi) using [these instructions](https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#1-overview). I did this entirely headlessly, without ever connecting the Pi to a monitor or keyboard. I plugged it directly into the router via ethernet, so there was no need to mess with wi-fi.

## SSH

I used SSH to connect to the Pi remotely. Note that the router configuration in these instructions was done on my CenturyLink Zyxel C3000Z, but with any luck the same steps can be translated to other routers.

1. Log into the router at http://192.168.0.1. Navigate to `Modem Status >> Device Table`. Find the Pi's MAC address that began with `DC:A6:32` (for a Raspberry Pi 4).
1. To assign a static IP address within the local network, navigate to `Advanced Setup >> DHCP Reservation`, under `4. Select MAC Address` select the MAC address or past it in manually, then select the desired IP address and click `Apply`. I assigned it to `192.168.0.3`.
1. To give the Pi a human readable name on the local network, navigate to `Advanced Setup >> DNS Host Mapping`. I set this up with:
    - DNS Host Name: `rphs` (**R**aspberry **P**i **H**ome **S**erver)
    - IP Address: `192.168.0.3`
    - Support Domain Name: `Disable` (not sure what this does, so I left it on the default)
1. To SSH into the Pi, run `ssh ubuntu@rphs`.

## Upgrade all Packages

To upgrade all packages, run the following.
```bash
sudo apt update && sudo apt upgrade --yes
```

The `update` will fetch the most recent version lists. Then `upgrade` will actually install them. The `--yes` flag will automatically accept the annoying prompt that asks you to confirm after the amount of space is calculated.

Note that `apt` is a newer, more user friendly alternative to `apt-get`.

## FSTAB: Mounting an External Hard Drive

I followed [these instructions](./pluralsight_home_server_course_notes.md#adding-external-storage) to add automatically mount my HDD via the file system table (fstab) config. I skipped the instructions for formatting the HDD, and instead opted to do it via the GParted GUI on an Ubuntu live USB on my MacBook. [This guide](https://help.ubuntu.com/community/Fstab) from Ubuntu was also useful.

```bash
# create a mount point
cd /mnt
sudo mkdir calaway_2tb
# Create a placeholder file to help see whether the mount point worked
# If you can see this file with `ls` then it is not mounted
sudo touch calaway_2tb/placeholder
# Find the UUID of the partition you wish to mount
sudo blkid | grep UUID
# Found `/dev/sda1: LABEL="calaway_2tb" UUID="ed6c1686-5997-424d-9a0e-fa6929a53431" TYPE="ext4" PARTUUID="572b7278-ab40-41df-b7a3-f2e0ea77e1a6"`
# Edit fstab
sudo vim /etc/fstab
```

I added the following line to the bottom of the fstab file.
```
UUID=ed6c1686-5997-424d-9a0e-fa6929a53431 /mnt/calaway_2tb ext4 defaults 0 0
```

```bash
# Reboot
sudo reboot
# Verify the partition loaded automatically by making sure the placholder file is not shown
ls /mnt/calaway_2tb
# Since my drive was newly formatted, for me this only listed `lost+found`
```

## Nextcloud

I opted to install [Nextcloud via snap](https://snapcraft.io/install/nextcloud/ubuntu), and found [this article from Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-nextcloud-on-ubuntu-20-04) to be useful.

```bash
sudo snap install nextcloud
```

After the installation is finished, navigate to the server over LAN at http://rphs. It will prompt you to set admin credentials.

## Dynamic DNS

I followed [this tutorial](https://engineerworkshop.com/blog/connecting-your-raspberry-pi-web-server-to-the-internet/) to set up DDNS.

```bash
# Install ddclient and go through the setup wizard
sudo apt install ddclient
# Make a copy of the original ddclient config file
sudo cp /etc/ddclient.conf{,.original}
# Edit the ddclient config
sudo vim /etc/ddclient.conf
```

```conf
# /etc/ddclient.conf

daemon=300
protocol=dyndns2
use=web
server=domains.google.com
ssl=yes
login=<username without quotes>
password='<password - KEEP SINGLE QUOTES>'
pi.my-site.com
```

```bash
# Make a copy and then edit the ddclient defaults
sudo cp /etc/default/ddclient{,.original}
sudo vim /etc/default/ddclient
```

```
# /etc/default/ddclient
# Comment out all lines except for these two:
run_daemon="true"
daemon_interval="300"
```

```bash
# Make sure that the ddclient service is running
sudo systemctl start ddclient
# Test our ddclient configuration with the following command:
sudo ddclient -daemon=0 -debug -verbose -noquiet
```

You should receive the following message:

```
SUCCESS:  engineerworkshop.com: skipped: IP address was already set
Going back to Google Domains and looking at your Dynamic DNS record should also show your public IP address.
```
