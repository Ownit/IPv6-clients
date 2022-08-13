# Debian 9

## Interfaces
```
enp3s0: WAN interface
enp2s0: LAN interface
```

## Install packages
`# apt install wide-dhcpv6-client radvd iptables-persistent`

## Set kernel parameters

#### /etc/sysctl.d/local.conf

```
net.ipv6.conf.enp3s0.accept_ra = 2
net.ipv6.conf.all.forwarding = 1
```

Reboot or run `sysctl --system` to apply the configuration.


## Configure wide-dhcpv6

#### /etc/default/wide-dhcpv6-client

```
INTERFACES="enp3s0"
```

#### /etc/wide-dhcpv6/dhcp6c.conf

```
interface enp3s0 {
    send ia-pd 1;
};

id-assoc pd 1 {
    prefix-interface enp2s0 {
        sla-id 0;
        sla-len 8;
        ifid 1;
    };
};
```

`systemctl restart wide-dhcpv6-client`


## Configure radvd

#### /etc/radvd.conf

```
interface enp2s0 {
    AdvSendAdvert on;
    MaxRtrAdvInterval 60;

    prefix ::/64 {
        AdvValidLifetime 300;
        AdvPreferredLifetime 120;
        AdvOnLink on;
        AdvAutonomous on;
    };
};
```

`systemctl restart radvd`


## ip6tables rules

#### /etc/iptables/rules.v6

```
# WAN: enp3s0
# LAN: enp2s0

*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]

-A INPUT -i enp2s0 -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -p icmpv6 -j ACCEPT
-A INPUT -i enp3s0 -p udp --dport 546 -j ACCEPT
-A INPUT -i enp3s0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i enp2s0 -o enp3s0 -j ACCEPT
-A FORWARD -i enp3s0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# Application rules
-A INPUT -i enp3s0 -p tcp --dport 22 -j ACCEPT

COMMIT
```

Reboot the server (the iptables-persistent package will help us apply /etc/iptables/rules.v6 on boot) or run `iptables-restore < /etc/iptables/rules.v6`


## nftables rules

Debian 10 (buster) and later use the nftables framework by default instead of iptables. More info is available [here](https://wiki.debian.org/nftables) at Debian Wiki. The example below should be similar to the iptables example, but nftables allows us to work with IPv4 and IPv6 at the same time through the *inet* family. The example also uses source NAT instead of masquerade for IPv4 since I always get the same external IP address from the DHCP-server.

#### /etc/nftables.conf

```
#!/usr/sbin/nft -f

# Define interfaces
define external = enp3s0
define internal = enp2s0

# External address for source NAT
define external_ip = 37.46.xxx.xxx

# Flush current ruleset
flush ruleset

table inet firewall {
        chain input {
                type filter hook input priority 0; policy drop;

                ct state invalid drop
                ct state established,related accept

                iif lo accept
                iif $internal accept

                ip protocol icmp limit rate 10/second accept
                ip6 nexthdr icmpv6 limit rate 10/second accept

                tcp dport 22 accept
                udp dport 546 ip6 daddr fe80::/64 accept
        }

        chain forward {
                type filter hook forward priority 0; policy drop;

                iif $external oif $internal ct state established,related accept
                iif $internal oif $external accept
        }

        chain output {
                type filter hook output priority 0; policy accept;
        }
}

table ip nat {
        chain postrouting {
                type nat hook postrouting priority 100; policy accept;

                oif $external snat $external_ip
        }
}
```

Use `nft -cf /etc/nftables.conf` to validate syntax and `nft -f /etc/nftables.conf` to apply the ruleset from file.
