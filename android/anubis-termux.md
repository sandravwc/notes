---
title: android/anubis-termux
---

- anubis (techaro) on rootless termux: static go binary, just runs

  ```sh
  curl -sL https://github.com/TecharoHQ/anubis/releases/download/v1.27.0/anubis-1.27.0-linux-arm64.tar.gz | tar xz
  cp anubis-*/bin/anubis ~/anubis/bin/
  # no tls in anubis. chain: haproxy (tls) -> anubis -> app, see [[android/haproxy-termux]]
  ```

- env (runit run script does `set -a; . env; set +a; exec anubis`)

  ```sh
  BIND=127.0.0.1:8923
  TARGET=http://127.0.0.1:8090
  METRICS_BIND=127.0.0.1:9090
  DIFFICULTY=4
  SERVE_ROBOTS_TXT=1
  COOKIE_SECURE=1
  USE_REMOTE_ADDRESS=0          # reads X-Real-IP from the lb instead
  # POLICY_FNAME=botPolicies.yaml to override the built-in policy, sample in the tarball doc/
  ```

- what the default policy does

  ```txt
  user agent contains Mozilla -> pow challenge, cookie 1 week
  known ai crawlers (GPTBot etc) -> explicit deny
  anything else (curl, scripts) -> pass through untouched: the challenge needs js, so the app must have its own auth
  decisions in the json log: "new challenge issued" / "explicit deny" with user_agent + path
  its own docs site 403s curl, read the tarball README / --help instead
  ```
