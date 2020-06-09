## Dynamic DNS

For setting up Dynamic DNS with Google Domains, I followed a combination of instructions from  [this](https://pimylifeup.com/raspberry-pi-port-forwarding/) Pi My Life Up tutorial, the [GitHub repo](https://github.com/ddclient/ddclient), and [Google's instructions](https://support.google.com/domains/answer/6147083?hl=en).

Install `ddclient` and dependencies as per the tutorial. The version from the Debian repository was 3.8.3, Google Domains support was added in 3.9.0, so I upgraded from the GitHub repo.

```bash
# Clone the repo.
git clone https://github.com/ddclient/ddclient.git
cd ddclient
# Checkout the latest tag.
git checkout v3.9.1
# Copy the original executable
sudo cp -v /usr/sbin/ddclient{,.original}
# Replace the executable with the version
sudo cp -v ddclient /usr/sbin/ddclient
# From the GitHub install instructions
mkdir /etc/ddclient
mkdir /var/cache/ddclient
# Make a copy of the original config
sudo cp -v /etc/ddclient.conf{,.original}
# Copy the sample configuration from the repo to the new directory, and make a copy of the original
sudo cp -v sample-etc_ddclient.conf /etc/ddclient/ddclient.conf
sudo cp -v sample-etc_ddclient.conf /etc/ddclient/ddclient.conf.original
# Edit the config file
sudo nano /etc/ddclient/ddclient.conf
```

Uncomment the following lines and fill in your own credentials and domain.
```
protocol=googledomains
login=generated_username
password=generated_password
your_resource.your_domain.tld
```

Restart the ddclient service.
```bash
sudo /etc/init.d/ddclient restart
```