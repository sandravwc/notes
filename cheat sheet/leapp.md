---
title: cheat sheet/leapp
tags: [cheat sheet]
---

- the thing itself
  ```sh
  dnf update -y && reboot
  dnf install -y http://repo.almalinux.org/elevate/elevate-release-latest-el$(rpm --eval %rhel).noarch.rpm
  dnf install leapp-upgrade leapp-data-almalinux
  leapp preupgrade
  leapp upgrade
  reboot
  ```
- get rid of junk pre upgrade
  ```sh
  #!/usr/bin/env bash
  
  sed -i '/^exclude=/d' /etc/dnf/dnf.conf
  dnf remove -y leapp.noarch leapp-data-almalinux.noarch leapp-deps-el8.noarch leapp-repository-deps-el8.noarch leapp-upgrade-el7toel8.noarch python2-leapp.noarch
  rm -f /etc/yum.repos.d/ELevate.repo
  update-crypto-policies --set LEGACY
  rm /etc/sysconfig/network-scripts/ifcfg-{eth0,lo}
  leapp preupgrade
  leapp answer --section check_vdo.confirm=True
  leapp preupgrade
  ```
- add new junk post upgrade
  ```sh
  #!/usr/bin/env bash
  monIp="192.168.10.9"
  sed -i '/^nagios/d' /etc/passwd
  sed -i '/^nagios/d' /etc/group
  
  dnf install --assumeyes \
    nagios-plugins \
    fprintd-pam
  
  dnf remove --assumeyes nagios-nrpe
  dnf install --assumeyes nagios-nrpe
  
  cat <<- NRPECONF > /etc/nagios/nrpe.d/000_allowed_hosts.cfg
  allowed_hosts=127.0.0.1,${monIp}
  NRPECONF
  
  cat <<- 'SUDOERS' > /etc/sudoers.d/50_nagios
  nagios          ALL=(ALL) NOPASSWD: /usr/lib/nagios/plugins/,/usr/lib/nagios/plugins/contrib/,/usr/lib64/nagios/plugins/,/usr/lib64/nagios/plugins/contrib/,/usr/local/libexec/nagios/
  Defaults:nagios !requiretty
  Defaults:nagios !env_keep
  SUDOERS
  
  chown root.root /etc/sudoers.d/50_nagios
  chmod 440 /etc/sudoers.d/50_nagios
  
  systemctl enable --now nrpe
  #nmcli c m lo ipv6.addresses "::1/128"
  #nmcli c m lo ipv6.method manual
  #nmcli c u lo
  update-crypto-policies --set DEFAULT:SHA1
  ```
- remove boomer netconf
  ```sh
  cat /etc/default/grub
    - Remove those flags: Ensure neither `net.ifnames=0` nor `biosdevname=0` is present.
  
  udevadm test-builtin net_id /sys/class/net/eth0 2>/dev/null
    - Look for ID_NET_NAME_PATH (e.g., enp0s3) or ID_NET_NAME_SLOT (e.g., ens192). This is your new device name.
    if shit is persistent af
   44191  2026-02-23 22:55:13 grep -r nomodeset
  44192  2026-02-23 22:55:16 ls -lsa
  44193  2026-02-23 22:56:24 grep -r nomodeset
  44194  2026-02-23 22:58:59 vi loader/entries/4d102664f4de429fa225bced22375db7-5.14.0-611.34.1.el9_7.x86_64.conf
  44195  2026-02-23 22:59:35 grubby --update-kernel=ALL --remove-args="nomodeset net.ifnames=0"
  44196  2026-02-23 22:59:40 ip addr addd 192.168.10.41/24 eth0
  44197  2026-02-23 22:59:50 ip addr add 192.168.10.41/24 eth0
  44198  2026-02-23 23:00:04 ip a
  44199  2026-02-23 23:00:18 ip addr add 192.168.10.41/24 dev eth0
  44200  2026-02-23 23:00:33 ip link set eth0 up
  44201  2026-02-23 23:00:38 ip route add default via 192.168.10.5 dev eth0
  44202  2026-02-23 23:00:39 cat /etc/re
  44203  2026-02-23 21:51:14 cat /etc/resolv.conf
  44204  2026-02-23 21:51:19 rm -rf /boot/*
  44205  2026-02-23 21:54:17 dnf reinstall -y kernel-core kernel-modules grub2-pc grub2-tools
  44206  2026-02-23 21:54:25 grub2-install /dev/sda
  44207  2026-02-23 21:54:34 grub2-mkconfig -o /boot/grub2/grub.cfg
  44208  2026-02-23 21:56:22 dracut -f
  44209  2026-02-23 21:56:30 grep "nomodeset" /boot/loader/entries/*.conf
  44210  2026-02-23 22:04:06 grep -r "nomodeset" /boot/
  44211  2026-02-23 22:52:51 reboot
  44212  2026-02-23 22:53:21 ip a
  ```
- custom internal repo (example layout for a company-internal package mirror)
  ```sh
  [company_extern_projects]
  name=Company Projects Packages (AlmaLinux_9)
  type=rpm-md
  baseurl=https://repo.example.internal/company:/extern:/projects/AlmaLinux_9/
  gpgcheck=1
  gpgkey=https://repo.example.internal/company:/extern:/projects/AlmaLinux_9/repodata/repomd.xml.key
  enabled=1
  
  [company-default]
  name=Company default repo (AlmaLinux_9)
  type=rpm-md
  baseurl=https://repo.example.internal/company:/extern/AlmaLinux_9/
  gpgcheck=1
  gpgkey=https://repo.example.internal/company:/extern/AlmaLinux_9/repodata/repomd.xml.key
  enabled=1
  ```
