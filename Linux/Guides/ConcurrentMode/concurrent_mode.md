---
sort: 1
---

This guide assumes that you have already set up an access point deamon with DHCP Serving enabled

If not, you can follow [this guide]({{ site.github.url }}/WiFi/Guides/AccessPointUsingHostapd)

## Configuration details (from original guide)

* The SoftAP needs to be active most of the time, since it needs to send beacons every **beacon_int** (default 100 Tu = 102 ms), with a DTIM present in the beacon every **dtim_interval** (1 DTIM sent every **beacon_int*****dtim_interval**)

* To avoid possible issues due to beacons from the SoftAP and from the AP being transmitted simultaneously, we recommend slightly changing the **beacon_int** in `hostapd.conf` file. A value of 107 or 113 is ok, since these are prime numbers just higher than 100 (considering that the external AP is using 100).

```
beacon_int=107
dtim_period=1
```

## Adding wlan1 to wlan0

```console
iw dev wlan0 interface add wlan1 type managed
```

In case you are starting Station on wlan1 before starting SoftAP on wlan0, also execute: `ip link set dev wlan0 down`

**Important Note**: `ip link set dev wlan0 down` is only needed when starting the Station on wlan1 before starting the SoftAP on wlan0, to avoid this error:

```
Failed to connect to non-global ctrl_ifname: wlan1 error: No such file or directory
```

## Connecting to a network using wpa_supplicant

Simplest method is to use `wpa_cli` that comes with `wpa_supplicant` package

1. Run : `wpa_cli`
2. `> scan`
3. `> scan_results`

    At this point you should get a list of available networks

4. `> add_network`
5. `> set_network 0 ssid "MYSSID"`
6. `> set_network 0 psk "passphrase"`
7. `> enable_network 0`
8. `> save_config`
9. `> quit`

At this point you should be connected :

```console
pi@raspberrypi:~ $ ip a

...

5: wlan1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0d:6f:73:93:6a brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.70/24 brd 192.168.1.255 scope global dynamic noprefixroute wlan1
       valid_lft 3593sec preferred_lft 3143sec
    inet6 2a01:e0a:1fa:f170:f1c4:c242:b971:4cde/64 scope global dynamic mngtmpaddr noprefixroute
       valid_lft 86380sec preferred_lft 86380sec
    inet6 fe80::3932:5c26:3ddb:17c4/64 scope link
       valid_lft forever preferred_lft forever
```

## Making it all automatic upon startup 

Silicon Labs provides a few sample scripts to perform such a task nicely. In this case we will simply use rc.local :

Add the following line to `/etc/rc.local` :

```
sleep 5
iw dev wlan0 interface add wlan1 type managed
sleep 2
```

If you had previously registered a network using `wpa_cli`, your device should connect automatically using `wlan1`.

