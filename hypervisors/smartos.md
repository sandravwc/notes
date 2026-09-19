---
title: hypervisors/smartos
---

- update nic for carp/vrrp
  ```sh
  echo '{"update_nics":[{"mac": "c2:fd:b1:a8:a1:43", "allow_ip_spoofing": true, "allow_mac_spoofing": true, "allow_restricted_traffic": true, "allow_unfiltered_promisc": true...}]}' | vmadm update uuid
  ```
- checking resources
  ```sh
  # basic health checks
          - RAM: echo ::memstat |mdb -k 
          - ZFS: zpool status / zfs list
          - IO load: iostat -xn 1
          - show disks: iostat -eE / diskinfo
          - hardware: sysinfo
          - top alternative: prstat -Z
  
  # network settings
          - /usbkey/config
  
  # crontab
          - 1. make changes (for example /opt/custom/crontab/foo.crontab)
            2. svcadm restart crontab
            3. rm /etc/cron.d/ixcron
            4. svcadm clear crontab
            5. svcs -a |grep crontab
  
  # postboot / rc.local
    - mkdir /opt/custom/bin
          - echo "bla cmd" >> /opt/custom/bin/postboot
          - svcadm enable postboot (only once)
  ```
