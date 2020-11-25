# rpi-access-point-no-bridge

This is a guide to setup an access point on your Raspberry Pi. The pi is connected to the internet via ethernet and is broadcasting the access point over wifi.

I am by no means an expert - in fact I did a lot of research from multiple sources - this document is a culmination of what worked for my RPi 3B+.

## Other Solutions

- Below is a list of sites that I tried and the possible reason why it failed:
  - [thepi.io: How to use your raspberry pi as a wireless access point](https://thepi.io/how-to-use-your-raspberry-pi-as-a-wireless-access-point/)
  - [rpi official: Setting up a Raspberry Pi as a routed wireless access point](https://www.raspberrypi.org/documentation/configuration/wireless/access-point-routed.md)
  - [sparkfun: Setting up a Raspberry Pi 3 as an Access Point](https://learn.sparkfun.com/tutorials/setting-up-a-raspberry-pi-3-as-an-access-point/all) - Uses hotplug which is outdated
  - [SurferTim GitHub post: Setting up a Raspberry Pi as an access point in a standalone network](https://github.com/SurferTim/documentation/blob/6bc583965254fa292a470990c40b145f553f6b34/configuration/wireless/access-point.md)

- Below are a list of solutions that I have yet to try, but seem good:
  - [RaspAP](https://raspap.com/). Haven't used it, but it seems to be a community wide solution(?)
  - [Pi AP](https://github.com/f1linux/pi-ap). Haven't used it, but it has more details about debugging and provides a modular approach

## ToDo

- [ ] Try creating an [alias](https://www.raspberrypi.org/documentation/configuration/wireless/access-point-routed.md) to ssh into the pi e.g. `address=/gw.wlan/192.168.4.1`
- [ ] Try setting up [RaspAP with PiHole](https://discourse.pi-hole.net/t/raspap-pihole/14739)
- [ ] Add common mistakes and how to debug issues - the most usefult terminal commands were: `sudo reboot` and `journalctl -xe`
- [x] Remove redundant commands
- [x] Add better explanations
- [x] Add further details/ links
- [x] Add sources

## Setting up a Raspberry Pi as an access point in a standalone network
### Setup

Use the following to [update](https://www.raspberrypi.org/documentation/raspbian/updating.mdhttps://www.raspberrypi.org/documentation/raspbian/updating.md) your Raspbian installation:
```
sudo apt update
sudo apt full-upgrade
```
Install all the required software:
```
sudo apt-get install dnsmasq hostapd
```
Since the configuration files are not ready yet, turn the new software off:
```
sudo systemctl stop dnsmasq hostapd
```

### Configuring a static IP

The Raspberry Pi needs to have a static IP address, so that connected devices know where the server is.
This documentation assumes that we are using the standard 192.168.x.x IP addresses for our wireless network, so we will assign the server the IP address 192.168.4.1. 
The `4` is arbitrary, but has been set to an unused set of addresses.
It is also assumed that the wireless device being used is `wlan0` (which can be check using `ifconfig`). Normally the dhcpcd daemon (DHCP client) will search the network for a DHCP server to assign a IP address to `wlan0`. This is disabled by editing the configuration file:

```
sudo nano /etc/dhcpcd.conf
```

Add
```
interface wlan0
  static ip_address=192.168.4.1/24
  nohook wpa_supplicant
```

### Configuring the DHCP server (dnsmasq)

The DHCP service is provided by dnsmasq. 
By default, the configuration file contains a lot of information that is not needed, and it is easier to start from scratch. 
Rename this configuration file, and edit a new one:

```
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig  
sudo nano /etc/dnsmasq.conf
```

Add the following and save it:

```
interface=wlan0      # Use the require wireless interface - usually wlan0
  dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
```

So for `wlan0`, we are going to provide IP addresses between 192.168.4.2 and 192.168.4.20, with a lease time of 24 hours. 
There are many more options for [dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html).

### Configuring the access point host software (hostapd)

You need to edit the hostapd configuration file to add the various parameters for your wireless network. 
After initial install, this will be a new/empty file.

```
sudo nano /etc/hostapd/hostapd.conf
```

Add the information below to the configuration file. 
This configuration assumes we are using channel `7`, with a network name of `pi`, and a password `RasPI@AP`.
To setup a 5 Ghz connection, the model should be a pi 3B + or above. Make sure to change the channel and hw_mode
Note that the name and password should **not** have quotes around them, and should have a minimum of 8 characters.

```
country_code=GB
interface=wlan0
ssid=pi
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=RasPI@AP
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

We now need to tell the system where to find this configuration file.

```
sudo nano /etc/default/hostapd
```

Find the line with `#DAEMON_CONF`, and replace it with this:

```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

### Start it up

Now restart the services:

```
sudo service dhcpcd restart
sudo systemctl unmask hostapd
sudo service hostapd start  
sudo service dnsmasq start  
```

The network should now be broadcasting from the Raspberry Pi with the SSID and password sepcified, and other devices can connect to it.
Connected devices can access the Raspberry Pi access point via its IP address for operations such as `rsync`, `scp`, or `ssh pi@192.168.4.1`.
However, internet access has not been configured.


## Using the Raspberry Pi as an access point to share an internet connection

### Setup traffic forwarding

To forward the traffic from wlan0 to eth0:

```
sudo nano /etc/sysctl.conf
```

Uncomment this line:
```
#net.ipv4.ip_forward=1
```

It should now read:
```
net.ipv4.ip_forward=1
```

## Add iptables rule

To add ip masquerading for outbound traffic on eth0 using iptables:

```
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

save the rules
```
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```

To load this on boot
```
sudo nano /etc/rc.local
```

Add this line just above `exit 0`:

```
iptables-restore < /etc/iptables.ipv4.nat
```

Reboot
```
sudo systemctl reboot
```
