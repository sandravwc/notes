---
title: cheat sheet/haproxy
---

- config check, foreground for a supervisor

  ```sh
  haproxy -c -f cfg
  haproxy -W -db -f cfg        # master-worker, no daemonize
  ```
