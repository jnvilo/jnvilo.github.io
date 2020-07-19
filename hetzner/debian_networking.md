## Debian config file

```
auto lo
iface lo inet loopback

iface lo inet6 loopback

auto enp3s0
iface enp3s0 inet static
	address 85.10.197.133/27
	gateway 85.10.197.129
	up route add -net 85.10.197.128 netmask 255.255.255.224 gw 85.10.197.129 dev enp3s0
# route 85.10.197.128/27 via 85.10.197.129

	
iface enp3s0 inet6 static
	address 2a01:4f8:a0:5466::2/64
	gateway fe80::1

auto vmbr0
iface vmbr0 inet static
	address 10.250.250.1/24
	bridge-ports none
	bridge-stp off
	bridge-fd 0

auto vmbr1
iface vmbr1 inet static
	address 10.250.251.1/24
	bridge-ports none
	bridge-stp off
	bridge-fd 0

        post-up echo 1 > /proc/sys/net/ipv4/ip_forward
	post-up iptables -t nat -A POSTROUTING -s '10.250.250.0/24' -o enp3s0 -j MASQUERADE
        post-up iptables -t nat -D POSTROUTING -s '10.250.250.0/24' -o enp3s0 -j MASQUERADE



post-up ip addr add 85.10.197.147/32 dev enp3s0
```

Adding the second IP is just added as a post-up command

