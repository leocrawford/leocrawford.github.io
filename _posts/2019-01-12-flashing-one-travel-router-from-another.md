layout: post
title: "Using One Travel Router to Flash Another"
categories: technology

---

I have two travel routers both running OpenWRT connected to each other via an ethernet cable, and on
when the second router starts it pings 192.168.1.2 to see if it alive, and if it
is it downloads a new flash image from a TFTP server at the same address.

Running the TFTP server is simple following [these instructions](https://github.com/alghanmi/openwrt_netgear-wndr3700/wiki/TFTP-Server-on-Your-OpenWRT-Router)

The challenge though is that I don't know what IP address I might be connected to or from, and the most likely source value is 192.169.1.1 which is also my wireless network router
(which is how I connect to travel both my routers)

Some research showed that it is possible to listen for "anyIP" in a subnet using something like this:

    ip route add local 192.168.1.0/24 dev lo

However as this clashes with my wireless router I had to pop it into a table and use IP rules
to treat traffic from br-lan differently. My first attempts to achieve this failed in ways I [still don't understand](https://superuser.com/questions/1504888/route2-anyip-fails-when-not-in-local-table).

In testing this I leaned about `rp_filter`, `arp_filter`, `route_localnet` and `src_valid_mark`, all of which I ended up setting thus:

    sysctl -w net.ipv4.conf.all.arp_filter=0
    sysctl -w net.ipv4.conf.all.rp_filter=0
    sysctl -w net.ipv4.conf.all.route_localnet=1
    sysctl -w net.ipv4.conf.all.src_valid_mark=1

Even with these changes it didn't work so I reverted to my old friend DNAT (in conjunction
with the ip route `ip route add local 192.168.1.0/24 dev lo table 100` and `sysctl` settings above).

    iptables -t nat -A PREROUTING -d 192.168.1.0/24 -i br-lan -j DNAT --to-destination 127.0.0.1

This certainly seemed to get the packets processed, but no responses. Using marks I can make it work (the order is important):

    iptables -t mangle -A OUTPUT -j CONNMARK --restore-mark --nfmask 0xffffffff --ctmask 0xffffffff
    iptables -t mangle -I POSTROUTING  -m conntrack --ctstate DNAT --ctorigdst 192.168.1.0/24 -j MARK --set-xmark 0x64/0xffffffff
    iptables -t mangle -A POSTROUTING -j CONNMARK --save-mark --nfmask 0xffffffff --ctmask 0xffffffff

I was under the impression that DNAT should work in reverse, but it dindidn't seem to. Maybe because we're using multiple rule tables.

For now I have decided to use the uBoot feature to use dhcp to accept an IP address, which seems to work just fine.
