---
title: android/haproxy-termux
---

- haproxy rootless in termux

  ```sh
  pkg install haproxy          # 3.4 as of 2026-09, flags: [[cheat sheet/haproxy]]
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

- runit run script, config dir: every *.cfg in name order, one backend file per app repo symlinked in

  ```sh
  #!/data/data/com.termux/files/usr/bin/sh
  exec 2>&1
  exec haproxy -W -db -f $HOME/haproxy.d
  # ~/haproxy.d/00-base.cfg  global/defaults/frontends (box level)
  # ~/haproxy.d/10-app.cfg -> ~/app/repo/deploy/haproxy.cfg   "backend app" only
  ```

- one wildcard cert, backend by subdomain, no per-app acl

  ```txt
  frontend https
      ...
      use_backend %[req.hdr(host),lower,field(1,.)]    # shoko.poco.example -> backend shoko; unknown/bare host -> default
      default_backend mealprep
  ```

  ```sh
  acme.sh --issue --server letsencrypt --dns dns_autodns -d poco.example -d '*.poco.example'
  acme.sh --install-cert -d poco.example --ecc --fullchain-file ... --key-file ... --reloadcmd "sh install-cert.sh"   # --issue alone does not reinstall
  # subdomains as CNAME -> poco, dyndns keeps rewriting one A record
  ```

- basic auth

  ```sh
  # termux build has no crypt(3): "encrypted passwords will not work" -> insecure-password only, so the userlist goes in a
  # non-repo file in the config dir (~/haproxy.d/05-auth.cfg, 0600). userlist must be loaded before the backend that references it (name order)
  userlist monitoring
      user mrk insecure-password <pw>
  backend prom
      http-request auth realm prometheus unless { http_auth(monitoring) }
  ```
