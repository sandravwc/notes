---
title: cheat sheet/pfsense
tags: [cheat sheet]
---

- fix pfSense-upgrade
  ```sh
  certctl rehash
  pkg-static clean -ay
  pkg-static update -f
  pkg-static install -fy pkg pfSense-repo pfSense-upgrade
  pfSense-upgrade -d
  ```
