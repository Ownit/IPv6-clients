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
