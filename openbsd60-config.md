# Tested on OpenBSD 6.0-beta

# Requirements:

pkg: wide-dhcpv6

`# pkg_add -v wide-dhcpv6`

### dhcp6c

Run dhcp6c with the -Df flag to debug and run in foreground.

`dhcp6c -Df em0`

# Config

## # sysctl
/etc/sysctl.conf:
=====

`net.inet6.ip6.forwarding=1`

## # hostname.if

Set in hostname.if or manually with ifconfig.

`ifconfig em0 inet6 autoconf`

/etc/hostname.em0
=====
```
dhcp
group wan
inet6 autoconf
```

## # DHCP6

/etc/dhcp6c.conf
=====
```
interface em0 {
    send ia-na 1;
    send ia-pd 1;
};

id-assoc na 1 { };

id-assoc pd 1 {
    prefix-interface vlan10 {
        sla-id 0; # id of the prefix to set on the interface 
        sla-len 8;
    };
};
```

## # rtadvd.conf

/etc/rc.conf.local
=====

Enable rtadvd for vlan10
```
rtadvd_flags="vlan10" 
```

/etc/rtadvd.conf
=====
``` 
vlan10:\
  :maxinterval=10:\
  :mininterval=5:\
  :dnssl="int.mydomain.example":
```
## # PF

/etc/pf.conf
=====
```
# Not a full PF ruleset.
# em0 is in group 'wan'

# Allow RA from remote router.
pass in log on wan inet6 proto icmp6 icmp6-type { routeradv neighbrsol }

# Allow DHCPv6
pass in log on wan inet6 proto udp to (wan) port 546

# Allow clients on vlan10 to communicate with router
pass in log on vlan10 inet6 proto icmp6 icmp6-type { echoreq routersol neighbradv neighbrsol }
```

### links to manpages
[rtadvd.conf](http://man.openbsd.org/OpenBSD-current/man5/rtadvd.conf.5)  
[hostname.if](http://man.openbsd.org/OpenBSD-current/man5/hostname.if.5)  
[pf.conf](http://man.openbsd.org/OpenBSD-current/man5/pf.conf.5)  
