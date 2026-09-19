---
title: android/haproxy-termux
---

- haproxy rootless in termux

  ```sh
  pkg install haproxy          # 3.4 as of 2026-09
  haproxy -c -f cfg            # config check
  haproxy -W -db -f cfg        # foreground master-worker, for runit
  # ports > 1024 only, no root. bind :8443 not :443, router forwards 443 -> 8443 if wanted
  # cert = one pem, chain + key concatenated. acme.sh reloadcmd:
  cat fullchain.pem key.pem > haproxy.pem && chmod 600 haproxy.pem && SVDIR=$PREFIX/var/service sv restart haproxy
  ```

- minimal tls terminator in front of a plain http app

  ```txt
  global
      log stdout format raw local0 info
      ssl-default-bind-options ssl-min-ver TLSv1.2
  defaults
      mode http
      option httplog
      option forwardfor
      timeout connect 5s
      timeout client 60s
      timeout server 900s
  frontend https
      bind :8443 ssl crt /path/haproxy.pem alpn h2,http/1.1
      http-request set-header X-Forwarded-Proto https
      http-request set-header X-Real-IP %[src]
      http-response set-header Strict-Transport-Security "max-age=31536000"
      default_backend app
  backend app
      server app 127.0.0.1:8090 check
  frontend stats
      bind <lan ip>:8404
      stats enable
      stats uri /
  ```

- runit run script

  ```sh
  #!/data/data/com.termux/files/usr/bin/sh
  exec 2>&1
  exec haproxy -W -db -f $HOME/repo/deploy/haproxy.cfg
  ```
