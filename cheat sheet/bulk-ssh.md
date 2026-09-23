---
title: cheat sheet/bulk-ssh
---

- change one config on every host matching a pattern, host list parsed out of motd

  ```sh
  awk '!/^#/ && /k8s[2-3]?-(node|master)/ {print $2}' /etc/motd | xargs -I % ssh -n % "sed -i '/zombie/ s/-w 5 -c 10/-w 40 -c 50/' /etc/nagios/nrpe_local.cfg"
  ```
