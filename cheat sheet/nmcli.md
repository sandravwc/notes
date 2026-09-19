---
title: cheat sheet/nmcli
---

- create con

  ```sh
  nmcli c a type ethernet \
    con-name ens193 \
    ifname ens192 \
    ipv4.me manual \
    ipv4.addresses 192.168.10.46/24 \
    ipv4.g 192.168.10.5 \
    ipv4.dns "192.168.10.5" \
    ipv6.me auto \
    ipv6.addresses "2001:4178:2:269:192:168:67:46/64"
  ```

- mod con

  ```sh
  nmcli c m "4e4c5597-f957-3d99-9d1d-be128163e829" connection.id "ens19"
  nmcli c m ens19 ipv4.addresses "192.168.11.231/24"
  nmcli c m ens19 -ipv4.addresses "192.168.11.231/24"
  nmcli c m ens19 ipv4.addresses "192.168.11.231/23"
  nmcli c m ens19 ipv4.gateway "192.168.11.1"
  nmcli c m ens19 ipv4.dns "1.1.1.1 8.8.8.8"
  nmcli c m ens19 ipv4.method manual
  nmcli c m ens19 connection.autoconnect yes
  ```

- add vlan

  ```sh
  nmcli con add type vlan con-name bond0.40 dev bond0 id 40 ip4 192.168.14.239/24
  ```

- restart connection

  ```sh
  nmcli c up ens19
  ```

- add route

  ```sh
  nmcli c m "Vlan bond0.253" +ipv4.routes "10.0.3.90/32 192.168.15.14"
  nmcli c up "Vlan bond0.253"
  ```

-
