# rpi-access-point-no-bridge

## Setting up a Raspberry Pi as an access point in a standalone network

## TODO
- Remove redundant commands
- Add better explanations
- Add further details/ links
- Add sources
- Add common mistakes and how to debug issues

### Setup

Use the following to update your Raspbian installation:
```
sudo apt-get update
sudo apt-get full-upgrade
```
Install all the required software in one go with this command: 
```
sudo apt-get install dnsmasq hostapd
```
Since the configuration files are not ready yet, turn the new software off as follows: 
```
sudo systemctl stop dnsmasq hostapd
```

### Configuring a static IP

We are configuring a standalone network to act as a server, so the Raspberry Pi needs to have a static IP address assigned to the wireless port. 
This documentation assumes that we are using the standard 192.168.x.x IP addresses for our wireless network, so we will assign the server the IP address 192.168.4.1. 
It is also assumed that the wireless device being used is `wlan0`.

First, the standard interface handling for `wlan0` needs to be disabled. Normally the dhcpcd daemon (DHCP client) will search the network for a DHCP server to assign a IP address to `wlan0`. This is disabled by editing the configuration file:

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

Type or copy the following information into the dnsmasq configuration file and save it:

```
interface=wlan0      # Use the require wireless interface - usually wlan0
  dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
```

So for `wlan0`, we are going to provide IP addresses between 192.168.4.2 and 192.168.4.20, with a lease time of 24 hours. 
If you are providing DHCP services for other network devices (e.g. eth0), you could add more sections with the appropriate interface header, 
with the range of addresses you intend to provide to that interface.
There are many more options for dnsmasq; see the [dnsmasq documentation](http://www.thekelleys.org.uk/dnsmasq/doc.html) for more details.

### Configuring the access point host software (hostapd)

You need to edit the hostapd configuration file, located at /etc/hostapd/hostapd.conf, to add the various parameters for your wireless network. 
After initial install, this will be a new/empty file.

```
sudo nano /etc/hostapd/hostapd.conf
```

Add the information below to the configuration file. 
This configuration assumes we are using channel 7, with a network name of NameOfNetwork, and a password AardvarkBadgerHedgehog. 
Note that the name and password should **not** have quotes around them, and should have a minimum of 8 characters.

```
country_code=GB
interface=wlan0
ssid=NameOfNetwork
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=AardvarkBadgerHedgehog
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

To setup a 5 Ghz connection, the model should be a pi 3B + or above. Make sure to change the channel and hw_mode


We now need to tell the system where to find this configuration file.

```
sudo nano /etc/default/hostapd
```

Find the line with #DAEMON_CONF, and replace it with this:

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

Using a wireless device, search for networks. The network SSID you specified in the hostapd configuration should now be present, and it should be accessible with the specified password.

If SSH is enabled on the Raspberry Pi access point, connect to it using the `pi` account:

```
ssh pi@192.168.4.1
```

By this point, the Raspberry Pi is acting as an access point, and other devices can associate with it. 
Associated devices can access the Raspberry Pi access point via its IP address for operations such as `rsync`, `scp`, or `ssh`.
However, the internet access has not been configured


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
