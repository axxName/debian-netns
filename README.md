# debian-netns

Some simple scripts to simplify configuring network namespaces on
Debian-like systems.  Copy them into the corresponding directories under
`/etc/network`.

To configure a veth pair into a namespace:
```
/etc/network/interfaces:

auto ns0
iface ns0 inet manual
    peer-netns myns
    peer-iface eth0
    post-down-delete-netns yes
    peer-links ns3eth1:eth1 ns3eth2:eth2 ns3eth3:eth3
```
Running `ifup ns0` will then create the `myns` network namespace and a veth
pair which joins `ns0` in the "real world" to `eth0` inside the namespace.

There are two more things you can specify:
```
/etc/network/interfaces:

auto ns0
iface ns0 inet manual
    peer-netns myns
    peer-iface eth0
    veth-mode l3
    configure-interfaces yes
    post-down-delete-netns yes
    post-up ip route add 1.2.3.0/24 dev ns0
    peer-links ns3eth1:eth1 ns3eth2:eth2 ns3eth3:eth3
```
`veth-mode l3` will configure the veth pair as a L3 point-to-point link
(meaning that you can then add routes with the next-hop set to the
interface, e.g., `ip route add 1.2.3.0/24 dev ns0`).

`configure-interfaces yes` will run `ifup -a` (or `ifdown -a`) inside the
`myns` namespace using interface config from `/etc/network/netns/myns/interfaces`.
```
/etc/network/netns/myns/interfaces:

auto eth0
iface eth0 inet static
       address 1.2.3.4
       netmask 255.255.255.0
       network 1.2.3.0
       dns-nameservers 8.8.8.8
       post-up ip route add default dev eth0

auto eth3
iface eth3 inet static
       address 11.22.33.44
       netmask 255.255.255.0
```

### Thoughts

Ideally, the namespace interface config would be under
`/etc/netns/myns/network/interfaces`, but thanks to this feature of `ip
netns exec`:

"ip netns exec automates handling of this configuration, file convention for
network namespace unaware applications, by creating a mount namespace and
bind mounting all of the per network namespace configure files into their
traditional location in /etc."

this means that when `ifup` runs, it won't see the `/etc/network/if-*.d`
scripts (at least if they're not mirrored in `/etc/netns/myns`).

It would be nice to have some better way to run `ifup` inside a namespace,
but this would require hacking `ifupdown`.  If you want to know how to run it
by hand, look in `if-up.d/netns`.  It's not as simple as you might think.

