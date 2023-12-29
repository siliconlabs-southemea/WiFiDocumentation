---
sort: 1
---

This guide assumes that you have already set up an access point deamon with DHCP Serving enabled 

If not, you can follow [this guide]({{ site.github.url }}/WiFi/Guides/AccessPointUsingHostapd)

# Installing And configuring IP Forwarding

## Enable forwarding 
This step only requires editing the linux sysctl.conf file by calling `nano /etc/sysctl.conf` :

Look for and uncomment the below line :

```
net.ipv4.ip_forward=1
```

## Set up and install NAT rules

Then make sure you have installed  `iptables` and `iproute2` packages :

``` console 
sudo apt-get update
sudo apt-get install iproute2 iptables
```

This will bring in `tc` which will allow us to set up traffic rules as well as NAT :

``` console
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
```

Make sure that you replace your interface names so it matches the ones from your system

Finally we will make those permanent by creating `/etc/iptables.ipv4.nat` which we will use upon startup to restore NAT:

``` console
sh -c "iptables-save > /etc/iptables.ipv4.nat"
```

We will use rc.d to set back these rules. Edit `/etc/rc.local` and add `iptables-restore < /etc/iptables.ipv4.nat` to it :

```
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

# Print the IP address
_IP=$(hostname -I) || true
if [ "$_IP" ]; then
  printf "My IP address is %s\n" "$_IP"
fi

iptables-restore < /etc/iptables.ipv4.nat

exit 0
```

Reboot and verify that IP forwarding is working fine by running a speedtest. 

If your ethernet interface and WAN are not the bottleneck in your setup, you will be able to keep the test results as a reference

Test setup and architecture used for writing this guide :

```
WAN (10G-BaseT) <==ETHERNET==> RaspberryPi 3 eth0(1G-BaseT) <==Forwarding==> RaspberryPi 3 wlan0(SDIO 25MHz WFx200) <==WiFi==> Samsung Galaxy S23 (speedtest.net)
```

Speedtest details :
```
DownStream tested using 36.5MB file
UpStream tested using 30.3MB file
```

Results were :

```
Downlink : 30.3 Mbps
Uplink : 23.2 Mbps
```

## Setting Up traffic control for WLAN clients 

Before jumping into traffic control, it is important to note that WiFi stations will first send data over the air according to 802.11. Traffic control occurs one step further in the AP.

802.11 Defines QoS in tis frames and priorities using 8bit ToS field, but all devices may not use this field properly (see qos_map_set in hostapd.conf). Hence this guide's focus on IP traffic management.

The goal we will try to achieve here is to share unevenly our WiFi IP bandwidth so that unwanted joining device cannot overflow the AP by dropping their traffic a step further. 

Several options are available on linux systems. In this guide `tc` is the utility from iproute2 that will help us maintain `wlan0` traffic shaped the way we want.

We will use the following scenario to set up our system :

* Our router acts as an ethernet range extender 
* Our router must accept 802.11 associations from up to 12 clients
* Our router must always keep room for up to 8 "VIP" clients (identified via MAC)
    * VIP clients will be guaranteed a bandwidth of up to 625kbps (kbits/sec)
    * Other clients will have to share the remaining
* Traffic control will be done at the IP level


### Automatically define IP and AP settings upon startup

To enable the above we will have to share some settings across services. Therefore we will define a script that will make this easier to set everything up. To do so we will use rc.local once more :


```console 
nano /etc/rc.local
```

Add the following lines to it, before `exit 0` :

```
# hostapd.conf editing
# Edit SSID
sed -i -s "s/^ssid=.*/ssid=WF200_QOS/" /etc/hostapd/hostapd.conf
# Edit max number of stations
sed -i -s "s/^max_num_sta=.*/max_num_sta=12/" /etc/hostapd/hostapd.conf

# dhcpcd.conf editing
# wlan0 static address edit, using , as regex delimitor
sudo sed -i -s "s,^[ \t]*static ip_address=.*,    static ip_address=192.168.5.254/24," /etc/dhcpcd.conf

# dnsmasq.conf editing
# wlan 0 dhcp range edit
sed -i -s "s/^dhcp-range=.*/dhcp-range=192.168.5.2,192.168.5.14,255.255.255.0,24h/" /etc/dnsmasq.conf

```

This will automatically go edit 3 services configuration files that should be present in your system :
* `/etc/hostapd/hostapd.conf`
* `/etc/dnsmasq.conf`
* `/etc/dhcpcd.conf`

It will rename our access point and limit its clients number to `12`, define our `wlan0` ip to `192.168.5.1` and define our IP range served to limit it to 12 as well

At this point we have a fully functional AP which :
* Does not accept more than 12 WiFi Stations
* Does not provide more than 12 IP Addresses to DHCP clients 
* Does not perform any traffic arbitration

### Disable all lines in rc.local

### Setting dnsmasq so it provides 2 ranges of addresses

The strategy we will be using to "discriminate" between known devices and other devices is traffic arbitration based on IP range.

We will set `dnsmasq` to distributes IP adresses depending on the MAC ones of the stations 

To do so we will edit `/etc/dnsmasq.conf` to create TAGs based on MAC addresses :

```
# We target wlan0 interface
interface=wlan0
# We deine the IP range, as well as DHCP Lease time
#dhcp-range=192.168.5.2,192.168.5.14,255.255.255.0,24h
dhcp-mac=set:known_mac,A4:75:B9:*:*:*
dhcp-range=tag:known_mac,192.168.5.2,192.168.5.10,255.255.255.0,24h
dhcp-range=192.168.5.129,192.168.5.132,255.255.255.0,24h
```

At this point 2 sets of addresses will be distributed, allowing us to perform IP based QOS

### Setting up traffic control on WLAN traffic

In order to achieve both way traffic control on both subnets we will be taking advantage of linux kernel module named ifb (Intermediate Functional Block)

By default the TC utility does not allow us to control traffic coming from WLAN clients (ingress) easily. Therefore we will redirect all ingress traffic towards an ifb interface on which we will be able to apply tc properly

***

Below is the TC architecture we will be using :

```
# wlan0 - Egress traffic going from WiFi AP to WiFi Clients
# 
# +----------+
# | root 1:  |
# +----------+
# |
# +-----------------------------------------------+
# | class 1:1 Overall Bandwidth set to 10 MBits/s |
# +-----------------------------------------------+
# |
# +----------------------+
# |1:10 Default Traffic  |
# +----------------------+
```

```
# ifb0 - Ingress traffic going from WiFi Clients to WiFi AP (mirrored from wlan0)
# 
# +----------+
# | root 1:  |
# +----------+
# |
# +-----------------------------------------------+
# | class 1:1 Overall Bandwidth set to 10 MBits/s |
# +-----------------------------------------------+
# |
# +----------------------+
# |1:10 Default Traffic  |
# +----------------------+
```

Since we measured a top 30 Mbits/s on our interface earlier, we will limit the overall interface traffic to 10Mbits/s each way

***

Below steps configure an ifb interface and mirrors all ingress traffic towards ifb0

We explicitly tell modprobe to set up only one interface (as it defaults to 2 otherwise)

```console
sudo modprobe ifb numifbs=1
sudo ip link set dev ifb0 up
sudo tc qdisc add dev wlan0 handle ffff: ingress
sudo tc filter add dev wlan0 parent ffff: protocol ip u32 match u32 0 0 action mirred egress redirect dev ifb0
```

At this point you should see a new interface appearing using `ip a`

Below steps shape traffic to WLAN clients (egress rules) according to above diagram

```console 
sudo tc qdisc add dev wlan0 root handle 1: htb default 10
sudo tc class add dev wlan0 parent 1: classid 1:1 htb rate 10mbit
sudo tc class add dev wlan0 parent 1:1 classid 1:10 htb rate 625kbit
```

Below steps shape traffic from WLAN clients (ingress mirrored as egress) according to above diagram

```console
sudo tc qdisc add dev ifb0 root handle 1: htb default 10
sudo tc class add dev ifb0 parent 1: classid 1:1 htb rate 10mbit
sudo tc class add dev ifb0 parent 1:1 classid 1:10 htb rate 512kbit
```

The architecture was the same as previously

```
WAN (10G-BaseT) <==ETHERNET==> RaspberryPi 3 eth0(1G-BaseT) <==Forwarding==> RaspberryPi 3 wlan0(SDIO 25MHz WFx200) <==WiFi==> Samsung Galaxy S23 (speedtest.net)
```

Speedtest details :
```
DownStream tested using 0.32MB file
UpStream tested using 0.75MB file
```

Results were :

```
Downlink : 0.57 Mbps
Uplink : 0.46 Mbps
```

### Dual way Traffic control depending on device origin

Now that we have set up traffic control, we will add one class that we will use to slow down even more traffic from undesired devices that may have joined our AP

```
# wlan0 - Egress traffic going from WiFi AP to WiFi Clients
# 
# +----------+
# | root 1:  |
# +----------+
# |
# +-----------------------------------------------+
# | class 1:1 Overall Bandwidth set to 10 MBits/s |
# +-----------------------------------------------+
# |
# +----------------------+  +----------------------+
# |1:10 Default Traffic  |  | 1:20 Unknown Traffic  
# +----------------------+  +----------------------+
```

```
# ifb0 - Ingress traffic going from WiFi Clients to WiFi AP (mirrored from wlan0)
# 
# +----------+
# | root 1:  |
# +----------+
# |
# +-----------------------------------------------+
# | class 1:1 Overall Bandwidth set to 10 MBits/s |
# +-----------------------------------------------+
# |
# +----------------------+  +----------------------+
# |1:10 Default Traffic  |  | 1:20 Unknown Traffic  
# +----------------------+  +----------------------+
```

On egress traffic, we will add class 1:20 and a filter that matches traffic to clients with IPs in the upper range of 192.168.5.0 . To clearly differentiate results we will also change known IPs limits

```console 
sudo tc class add dev wlan0 parent 1:1 classid 1:10 htb rate 2mbit ceil 2mbit

sudo tc class add dev wlan0 parent 1:1 classid 1:20 htb rate 256kbit ceil 256kbit
sudo tc filter add dev wlan0 parent 1: protocol ip prio 1 u32 match ip dst 192.168.5.128/25 flowid 1:20
```

On ingress traffic, we will do the same for traffic coming from the upper range of 192.168.5.0

```console
sudo tc class add dev ifb0 parent 1:1 classid 1:10 htb rate 2mbit ceil 2mbit

sudo tc class add dev ifb0 parent 1:1 classid 1:20 htb rate 256kbit ceil 256kbit
sudo tc filter add dev ifb0 parent 1: protocol ip prio 1 u32 match ip src 192.168.5.128/25 flowid 1:20
```

**Note : Even though we have not set 2 subnets but only defined a range with dnsmasq, tc will apply rules based solely on IP and not look for any subnet**

The architecture was the same as previously

```
WAN (10G-BaseT) <==ETHERNET==> RaspberryPi 3 eth0(1G-BaseT) <==Forwarding==> RaspberryPi 3 wlan0(SDIO 25MHz WFx200) <==WiFi==> Samsung Galaxy S23 (speedtest.net)
```

Speedtest details for an known IP:
```
DownStream tested using 1.24MB file
UpStream tested using 1.74MB file
```

Results were :

```
Downlink : 1.89 Mbps
Uplink : 1.86 Mbps
```

Speedtest details for an unknown IP:
```
DownStream tested using 0.19MB file
UpStream tested using 0.35MB file
```

Results were :

```
Downlink : 0.22 Mbps
Uplink : 0.22 Mbps
```

### Automating QOS setup at startup

Just like IP forwarding rules we will apply those setting at startup using a script called by RC. Just for the sake of this guide we will place it in /etc :

`nano /etc/tc.ipv4.wlan0`

Then copy paste the below

```
#!/bin/bash

# Get rid of any existing qdisc on wlan0
tc qdisc del dev wlan0 root 

modprobe ifb numifbs=1
ip link set dev ifb0 up
tc qdisc add dev wlan0 handle ffff: ingress
tc filter add dev wlan0 parent ffff: protocol ip u32 match u32 0 0 action mirred egress redirect dev ifb0

#Shape traffic to WLAN clients (egress rules)
tc qdisc add dev wlan0 root handle 1: htb default 10
tc class add dev wlan0 parent 1: classid 1:1 htb rate 10mbit
tc class add dev wlan0 parent 1:1 classid 1:10 htb rate 2mbit ceil 2mbit

tc class add dev wlan0 parent 1:1 classid 1:20 htb rate 256kbit ceil 256kbit
tc filter add dev wlan0 parent 1: protocol ip prio 1 u32 match ip dst 192.168.5.128/25 flowid 1:20


#Shape traffic from WLAN clients (ingress mirrored as egress)
tc qdisc add dev ifb0 root handle 1: htb default 10
tc class add dev ifb0 parent 1: classid 1:1 htb rate 10mbit
tc class add dev ifb0 parent 1:1 classid 1:10 htb rate 2mbit ceil 2mbit

tc class add dev ifb0 parent 1:1 classid 1:20 htb rate 256kbit ceil 256kbit
tc filter add dev ifb0 parent 1: protocol ip prio 1 u32 match ip src 192.168.5.128/25 flowid 1:20

```

Give execute permissions to the script using `chmod +x /etc/tc.ipv4.wlan0.sh`

And finally call it in rc.local :

`nano /etc/rc.local`

```
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

# Print the IP address
_IP=$(hostname -I) || true
if [ "$_IP" ]; then
  printf "My IP address is %s\n" "$_IP"
fi

# hostapd.conf editing
# Edit SSID
sed -i -s "s/^ssid=.*/ssid=WF200_QOS/" /etc/hostapd/hostapd.conf
# Edit max number of stations
sed -i -s "s/^max_num_sta=.*/max_num_sta=12/" /etc/hostapd/hostapd.conf

# dhcpcd.conf editing
# wlan0 static address edit, using , as regex delimitor
sudo sed -i -s "s,^[ \t]*static ip_address=.*,    static ip_address=192.168.5.254/24," /etc/dhcpcd.conf

# dnsmasq.con editing
# wlan 0 dhcp range edit
#sed -i -s "s/^dhcp-range=.*/dhcp-range=192.168.5.2,192.168.5.14,255.255.255.0,24h/" /etc/dnsmasq.conf

iptables-restore < /etc/iptables.ipv4.nat

/etc/tc.ipv4.wlan0.sh

exit 0
```
                                                                                  
You can now reboot and check that these new settings are applied 

## Evenly share bandwidth among devices of a same group

What we did before was somewhat brutal, but allows us to never overload the WiFi chipset by both unknown devices, as well as known devices 

We will re-work a few settings to allow our system to be more flexible 

### Changing DHCP Lease times

We used `dnsmasq` to provide 2 ranges of addresses. Each range has a limited number of leases that can be provided to clients 

However if clients end up leaving the network, `dnsmasq` will only wait until the end of the lease to free a slot

Which means that lease time must be wisely chosen to reduce latency when adding/removing devices around the range limit

To do so, we will simply change the lease time in `/etc/dnsmasq.conf` :

```
# We target wlan0 interface
interface=wlan0
# We deine the IP range, as well as DHCP Lease time
#dhcp-range=192.168.5.2,192.168.5.14,255.255.255.0,24h
dhcp-mac=set:known_mac,A4:75:B9:*:*:*
dhcp-range=tag:known_mac,192.168.5.2,192.168.5.10,255.255.255.0,2h
dhcp-range=192.168.5.129,192.168.5.132,255.255.255.0,30m
```

### Revising `tc` bandwidths

We applied Queue discpline on both egress and outgress traffic through ifb

In our case, assuming the WiFi chipset was the bottleneck, we measured ~30 Mbits/s datarate

According to Silicon Labs in [this post]() it might be possible to achieve up to 50Mbits/s

Previously we defined a top IP datarate of 20 MBits shared evenly between egress and ingress traffic

We will change that balance as we do not expect devices to generate upload besides periodic reports and/or acks. We will set a 80%-20% instead

Another point that we had done was to not use 100% of our 1:1 classes, which means all of our allowed devices had to share an extremely downsized bandwidth

Here again we will revise these and provide higher rates and ceil values, keeping the sum of ceils within the 1:1 rate 

This results in :
```
#!/bin/bash

# Get rid of any existing qdisc on wlan0
tc qdisc del dev wlan0 root 

modprobe ifb numifbs=1
ip link set dev ifb0 up
tc qdisc add dev wlan0 handle ffff: ingress
tc filter add dev wlan0 parent ffff: protocol ip u32 match u32 0 0 action mirred egress redirect dev ifb0

#Shape traffic to WLAN clients (egress rules)
tc qdisc add dev wlan0 root handle 1: htb default 10
tc class add dev wlan0 parent 1: classid 1:1 htb rate 16mbit
tc class add dev wlan0 parent 1:1 classid 1:10 htb rate 14mbit ceil 15mbit

tc class add dev wlan0 parent 1:1 classid 1:20 htb rate 512kbit ceil 625kbit
tc filter add dev wlan0 parent 1: protocol ip prio 1 u32 match ip dst 192.168.5.128/25 flowid 1:20


#Shape traffic from WLAN clients (ingress mirrored as egress)
tc qdisc add dev ifb0 root handle 1: htb default 10
tc class add dev ifb0 parent 1: classid 1:1 htb rate 4mbit
tc class add dev ifb0 parent 1:1 classid 1:10 htb rate 2mbit ceil 3mbit

tc class add dev ifb0 parent 1:1 classid 1:20 htb rate 512kbit ceil 625kbit
tc filter add dev ifb0 parent 1: protocol ip prio 1 u32 match ip src 192.168.5.128/25 flowid 1:20

```

With these new settings, we have our known devices sharing a 14Mbits/s Down - 2Mbits/s Up .
The remaining 4 devices allowed will share a much lower part of that : 512kbits/s Down - 512kbits/s Up.

### Revising `tc` policies to avoid blocking communications

Even though we just increased bandwidth for our 10 known devices (and 4 others), the Queue discipline algorithms may not be the best to ensure fair and complete packet distribution

The problem we may encounter with the previous classes is that the HTB algorithm used for our trafic shaping classes is mainly targetting traffic shaping, but not management.

Using SFQ (Stochastic Fairness Queuing) will allow us to evenly share bandwidth between connections after shaping traffic
Therefore we will enable SFQ Queue Disciplines to our leaf classes


This results in :
```
#!/bin/bash

# Get rid of any existing qdisc on wlan0
tc qdisc del dev wlan0 root 

modprobe ifb numifbs=1
ip link set dev ifb0 up
tc qdisc add dev wlan0 handle ffff: ingress
tc filter add dev wlan0 parent ffff: protocol ip u32 match u32 0 0 action mirred egress redirect dev ifb0

#Shape traffic to WLAN clients (egress rules)
tc qdisc add dev wlan0 root handle 1: htb default 10
tc class add dev wlan0 parent 1: classid 1:1 htb rate 16mbit
tc class add dev wlan0 parent 1:1 classid 1:10 htb rate 14mbit ceil 15mbit

tc class add dev wlan0 parent 1:1 classid 1:20 htb rate 512kbit ceil 625kbit
tc filter add dev wlan0 parent 1: protocol ip prio 1 u32 match ip dst 192.168.5.128/25 flowid 1:20

tc qdisc add dev wlan0 parent 1:10 handle 10: sfq perturb 10
tc qdisc add dev wlan0 parent 1:20 handle 20: sfq perturb 10

#Shape traffic from WLAN clients (ingress mirrored as egress)
tc qdisc add dev ifb0 root handle 1: htb default 10
tc class add dev ifb0 parent 1: classid 1:1 htb rate 4mbit
tc class add dev ifb0 parent 1:1 classid 1:10 htb rate 2mbit ceil 3mbit

tc class add dev ifb0 parent 1:1 classid 1:20 htb rate 512kbit ceil 625kbit
tc filter add dev ifb0 parent 1: protocol ip prio 1 u32 match ip src 192.168.5.128/25 flowid 1:20

tc qdisc add dev ifb0 parent 1:10 handle 10: sfq perturb 10
tc qdisc add dev ifb0 parent 1:20 handle 20: sfq perturb 10

```

### Hostapd config file 
A few other can be improved in such scenario

```
# Station inactivity limit
#
# If a station does not send anything in ap_max_inactivity seconds, an
# empty data frame is sent to it in order to verify whether it is
# still in range. If this frame is not ACKed, the station will be
# disassociated and then deauthenticated. This feature is used to
# clear station table of old entries when the STAs move out of the
# range.
#
# The station can associate again with the AP if it is still in range;
# this inactivity poll is just used as a nicer way of verifying
# inactivity; i.e., client will not report broken connection because
# disassociation frame is not sent immediately without first polling
# the STA with a data frame.
# default: 300 (i.e., 5 minutes)
#ap_max_inactivity=300
```