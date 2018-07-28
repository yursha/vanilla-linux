# Install host system

The system is loaded with SeaBios.

## Configure the network

### Find out the name of your wireless network interface

```
sh> ls /sys/class/net
lo wlp1s0
```

`wlp1s0` is a wireless network interface, so we can be sure that the device driver was loaded successfully.

### Enable the interface

Check the status of the interface `wlp1s0`.

```
sh> ip link show dev wlp1s0
2: wlp1s0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
  link/ether 6c:29:95:a6:85:97 brd ff:ff:ff:ff:ff:ff
```

The interface is disabled because `<BROADCAST,MULTICAST>` misses `UP` in it.

Enable the interface.

```
sh> ip link set wlp1s0 up
sh> ip link show dev wlp1s0
2: wlp1s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
  link/ether 6c:29:95:a6:85:97 brd ff:ff:ff:ff:ff:ff
```

### Connect to the network

We'll be using `wpa_supplicant (8)` (http://w1.fi/wpa_supplicant/) to connect to the Wi-Fi network. `wpa_supplicant` is Wi-Fi Protected Access client and IEEE 802.1X supplicant.

WPA stands for W.
