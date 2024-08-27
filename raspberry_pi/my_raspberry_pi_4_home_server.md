## Install Ubuntu Server

### 2024 Update

I followed [these instructions](https://ubuntu.com/download/raspberry-pi) to install Ubuntu server 24.04 LTS via the Raspberry Pi Imager. Leverage the custom config settings on the Raspberry Pi Imager to automatically connect to your wi-fi and have your SSH key installed. I was able to SSH into the pi immediately after booting it up the first time without ever connecting to a monitor.

### 2020 Original

I installed Ubuntu 20.04 LTS (64-bit) from [here](https://ubuntu.com/download/raspberry-pi) using [these instructions](https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#1-overview). I did this entirely headlessly, without ever connecting the Pi to a monitor or keyboard. I plugged it directly into the router via ethernet, so there was no need to mess with wi-fi.

## SSH

### 2024 Update

I found the Pi on my UniFi Dream Router under Network >> Client Devices and logged in via `ssh calaway@192.168.1.9`. The router also automatically applied the host name, so I was able to log in without looking up the local IP via `ssh calaway@rphs`.

### 2020 Original

I used SSH to connect to the Pi remotely. Note that the router configuration in these instructions was done on my CenturyLink Zyxel C3000Z, but with any luck the same steps can be translated to other routers.

1. Log into the router at http://192.168.0.1. Navigate to `Modem Status >> Device Table`. Find the Pi's MAC address that began with `DC:A6:32` (for a Raspberry Pi 4).
    - Alternatively, you can find the IP address by following the `nmap` instructions [here](https://www.raspberrypi.org/documentation/remote-access/ip-address.md):
    - `brew install nmap`
    - `sudo nmap -sn 192.168.0.0/24`
1. To assign a static IP address within the local network, navigate to `Advanced Setup >> DHCP Reservation`, under `4. Select MAC Address` select the MAC address or past it in manually, then select the desired IP address and click `Apply`. I assigned it to `192.168.0.3`.
1. To give the Pi a human readable name on the local network, navigate to `Advanced Setup >> DNS Host Mapping`. I set this up with:
    - DNS Host Name: `rphs` (**R**aspberry **P**i **H**ome **S**erver)
    - IP Address: `192.168.0.3`
    - Support Domain Name: `Disable` (not sure what this does, so I left it on the default)
1. To SSH into the Pi, run `ssh ubuntu@rphs`.

### RSA Authentication

To require asymetric key authentication, I followed a combination of instructions from [my PluralSight notes](./pluralsight_home_server_course_notes.md#remote-duplicati) and this [PiMyLifeUp article](https://pimylifeup.com/raspberry-pi-ssh-keys/).

To generate a key pair, I recommend following the [instructions on GitHub](https://help.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh).

Copy the public key from my MacBook.
```bash
pbcopy < ~/.ssh/id_rsa.pub
```

Save the public key on the Pi.
```bash
# Backup the original
cp -v ~/.ssh/authorized_keys{,.original}
# Edit the config, paste in the key at the end of the file, and save
vim ~/.ssh/authorized_keys
```

Exit your `ssh` session and start a new one. Verify you can connect without a password. **Do not proceed if this does not work, or you will lock yourself out of the Pi.** (You would at least be able to log in via by connecting a monitor and keyboard to the Pi directly.)

Disable password logins.
```bash
# Make a copy of the sshd config
sudo cp -v /etc/ssh/sshd_config{,.original}
# Edit the config
sudo vim /etc/ssh/sshd_config
# Restart the SSH daemon to pick up the config change
sudo service sshd restart
```

Uncomment and edit the PasswordAuthentication line to be
```
PasswordAuthentication no
```

## Upgrades

### Manually Upgrade All Packages
To upgrade all packages, run the following.
```bash
sudo apt update && sudo apt upgrade --yes
```

The `update` will fetch the most recent version lists. Then `upgrade` will actually install them. The `--yes` flag will automatically accept the annoying prompt that asks you to confirm after the amount of space is calculated.

Note that `apt` is a newer, more user friendly alternative to `apt-get`.

### Automatically Upgrade All Packages
Follow [these instructions](https://help.ubuntu.com/community/AutomaticSecurityUpdates#Using_the_.22unattended-upgrades.22_package) to enable automatic upgrades.
```bash
# Install the thing
sudo apt-get install unattended-upgrades

# Enable the thing
sudo dpkg-reconfigure --priority=low unattended-upgrades

# Backup the original config
sudo cp /etc/apt/apt.conf.d/20auto-upgrades{,.original}

# Edit the config
sudo vim /etc/apt/apt.conf.d/20auto-upgrades
```

Add the following line to automatically reboot when necessary:
```
Unattended-Upgrade::Automatic-Reboot "true";
```

### Upgrade Ubuntu Release

Documentation from Ubuntu can be found [here](https://ubuntu.com/server/docs/how-to-upgrade-your-release).

Pre-upgrade checklist:

1. Upgrade all packages (see above)
1. Verify that you have at least a few gigabytes of free disk space via `df -h`
1. Backup all data (see below)
1. Open backup port: `iptables -I INPUT -p tcp --dport 1022 -j ACCEPT`

```bash
# Do the thing
sudo do-release-upgrade
```

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

## Port Forwarding

Find the local IP address either by looking it up on your router's device table or by running `ifconfig | grep inet`.

Here's how I set up port forwarding on my CenturyLink Zyxel C3000Z, but with any luck the same steps can be translated to other routers.

1. Log into the router at http://192.168.0.1. Navigate to `Advanced Setup >> Port Forwarding`.
1. Section 1: Select the Pi from the dropdown, or manually enter IP address.
1. Section 2: Enter `80` for both the starting and ending ports.
1. Section 3: Select `TCP` for the protocol.
1. Section 4: Select `All IP Addresses`.
1. Section 5: Click `Apply`.
1. Repeat for port `443`.

## Dynamic DNS

I followed [this tutorial](https://engineerworkshop.com/blog/connecting-your-raspberry-pi-web-server-to-the-internet/) to set up DDNS.

```bash
# Install ddclient and go through the setup wizard
sudo apt install ddclient
# Make a copy of the original ddclient config file
sudo cp -v /etc/ddclient.conf{,.original}
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
sudo cp -v /etc/default/ddclient{,.original}
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

## Nextcloud

I opted to install [Nextcloud via snap](https://snapcraft.io/install/nextcloud/ubuntu), and found [this article from Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-nextcloud-on-ubuntu-20-04) to be quite useful.

#### Install

```bash
# Install via snap
sudo snap install nextcloud

# Manually set the login credentials
sudo nextcloud.manual-install <username> <password>
```


#### Set Trusted Domains

```bash
# View all current trusted domains
sudo nextcloud.occ config:system:get trusted_domains
# Set a second trusted domain in index 1
sudo nextcloud.occ config:system:set trusted_domains 1 --value=rphs
# Set a third trusted domain in index 2
sudo nextcloud.occ config:system:set trusted_domains 2 --value=pi.my-site.com
# Verify the updated list of trusted domains
sudo nextcloud.occ config:system:get trusted_domains
```

#### External Hard Drive

Follow the [wiki article](https://github.com/nextcloud/nextcloud-snap/wiki/Change-data-directory-to-use-another-disk-partition) to change data directory to use another disk partition. This [Stack answer](https://askubuntu.com/a/882696) was also helpful.

```bash
# Give the snap permission to access removable media by connecting that interface
sudo snap connect nextcloud:removable-media

# Verify the connection was added
snap connections nextcloud

# Create the directory and set owner & permissions
sudo mkdir -p /mnt/calaway_2tb/nextcloud/data
sudo chown -R root:root /mnt/calaway_2tb/nextcloud/data

# Stop the snap from running for a moment
 $ sudo snap stop nextcloud

# Make a copy and then edit the Nextcloud config
sudo cp -v /var/snap/nextcloud/current/nextcloud/config/config.php{,.original}
sudo vim /var/snap/nextcloud/current/nextcloud/config/config.php
```

```config
# Change
'datadirectory' => '/var/snap/nextcloud/common/nextcloud/data'
# to the desired directory
'datadirectory' => '/mnt/calaway_2tb/nextcloud/data'
```

```bash
# Copy the current data directory to the new place
sudo cp -rv /var/snap/nextcloud/common/nextcloud/data /mnt/calaway_2tb/nextcloud

# Re-enable the snap:
sudo snap start nextcloud
```

#### HTTPS via Let's Encrypt

```bash
# Run this script and follow the prompts
# (Reminder: I used email address e********2@gmail.com)
sudo nextcloud.enable-https lets-encrypt
```

## Tooling

### Oh My Zsh

Follow instructions from the [Oh My Zsh readme](https://github.com/ohmyzsh/ohmyzsh).

```bash
# Check to see if zsh is already installed
zsh --version
# If not, install it
sudo apt install zsh --yes
# Download and run the Oh My Zsh install script
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

Copy over anything from `~/.bashrc` that you want. Note that `shopt` isn't available in Zsh, so don't copy any of that. Anything with the prompt didn't work for me either.

Customizations:
* Theme: `ZSH_THEME="avit"`
* [Autosuggestions](https://github.com/zsh-users/zsh-autosuggestions/blob/master/INSTALL.md#oh-my-zsh)
* [Syntax highlighting](https://github.com/zsh-users/zsh-syntax-highlighting/blob/master/INSTALL.md#oh-my-zsh)
* [fasd](https://github.com/clvv/fasd/wiki/Installing-via-Package-Managers)

## Backup SD Card

I followed a mix of these instructions from [my PluralSight notes](./pluralsight_home_server_course_notes.md#backing-up) and [this article](https://www.linux.com/topic/desktop/full-metal-backup-using-dd-command/).

Power down the Pi.
```bash
sudo shutdown -h now
```

Remove the SD card and insert it into the machine you're backing it up to.
```bash
# Find the correct device
diskutil list
# Make a bit-for-bit copy and compress it
sudo dd if=/dev/diskx bs=1m conv=noerror,sync | gzip -c  > ~/Documents/backups/pi_2020-07-31.img.gz
```
