---
sort: 1
---

# Installing And configuring Host AP Daemon

First thing required is installing `hostapd` :

``` console 
sudo apt install hostapd
```

Once done edit the `hostapd.conf` file located in `/etc/hostapd/` as below :

``` 
# Example configuration file for hostapd
#
# The full documentation is available at https://w1.fi/cgit/hostap/plain/hostapd/hostapd.conf
#

interface=wlan0
driver=nl80211
ctrl_interface=/var/run/hostapd
ctrl_interface_group=netdev

ssid=YOUR_AP_SSID

country_code=FR
channel=1

auth_algs=1
hw_mode=g
ieee80211n=1

beacon_int=100
dtim_period=1

max_num_sta=16

# Uncomment to enable WPA2-PSK-CCMP authentication (WPA2-Personal)
#wpa=2
#wpa_passphrase=YOUR_WPA2_PASSPHRASE
#wpa_key_mgmt=WPA-PSK
#rsn_pairwise=CCMP
#ieee80211w=1

# Uncomment to enable WPA2-SAE-CCMP authentication (WPA3-Personal)
#wpa=2
#wpa_passphrase=YOUR_WPA3_PASSPHRASE
#wpa_key_mgmt=SAE
#rsn_pairwise=CCMP
#ieee80211w=2
```

Finally, enable the service to start upon linux boot and reboot:

```console
systemctl unmask hostapd
systemctl enable hostapd
reboot
```

You should now see a new access point `YOUR_AP_SSID` around

# Configuring DHCP client for wlan interface

In order to start accepting clients, we need to assign a static IP address to our wlan interface. Doing so requires to modify the DHCP Client daemon config file present in our linux system at `/etc/dhcpcd.conf`.

There we will add the following :

```
interface wlan0
    static ip_address=192.168.4.1/24
    nohook wpa_supplicant
```

# Installing and configuring DHCP server

At this point we need to enable clients to connect ang gather an IP adress as well

For that we will be using `dnsmasq`, so we will start by installing it :

```console
apt install dnsmasq
```

We edit its config file located in `/etc/dnsmasq.conf` with the below :

```console
# We target wlan0 interface
interface=wlan0
# We deine the IP range, as well as DHCP Lease time
dhcp-range=192.168.4.2,192.168.4.100,255.255.255.0,24h
```

***Make Sure the static IP address and range match***

Finally we enable the service :

```console
systemctl unmask dnsmasq
systemctl enable dnsmasq
reboot
```

