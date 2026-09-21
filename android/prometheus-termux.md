---
title: android/prometheus-termux
---

- prometheus + alertmanager + blackbox + node_exporter rootless in termux, repo `sandravwc/monitoring` (readme = install)
  - upstream linux-arm64 tarballs, no pkg. `deploy/fetch.sh <name> [ver]`, sv services, everything on 127.0.0.1
  - alerts -> ntfy: `https://ntfy.sh/<topic>?template=alertmanager`, ntfy formats the json itself, no bridge. url in a file (`url_file:`), not in the repo
  - dead man's switch: workstation systemd user timer curls prometheus over ssh, ntfy if down

- what android denies apps, and the side door for each

  ```sh
  /proc/stat /proc/loadavg /proc/vmstat /proc/uptime     # denied -> no cpu/load collectors. proot-distro fakes exactly these files
  # load: sysinfo() syscall works (bionic getloadavg does that). python ctypes: struct { long uptime; ulong loads[3]; ... } loads/65536
  # cpu busy: /sys/devices/system/cpu/cpu*/cpuidle/state*/time readable (us idle). busy = 1 - d(idle)/d(CLOCK_BOOTTIME)
  # compute the ratio in the cron script, not with rate(): file written 1/min, scraped 2/min -> plateaus, rate() under-counts idle
  /sys/class/thermal/thermal_zone*/temp                   # cpu* zones readable, others not -> node_exporter thermal_zone collector fails on the first denied zone, read cpu zones yourself
  netlink                                                 # no net metrics. alertmanager --cluster.listen-address="" (gossip wants an advertise ip)
  open_tree()                                             # seccomp SIGSYS. node_exporter >= 1.9 uses it (filepath-securejoin) -> dies on first scrape. pin 1.8.2
  /etc/resolv.conf /etc/ssl                                # missing. go's pure resolver -> "lookup on [::1]:53", port 53 unbindable so no local forwarder
  #   fix: tinyproxy (bionic, resolves like curl) on 127.0.0.1:8118, proxy_url in alertmanager http_config + blackbox module (+ skip_resolve_phase_with_proxy: true)
  #   scrape targets stay ips. SSL_CERT_FILE=$PREFIX/etc/tls/cert.pem in every run script
  ```

- node_exporter that survives

  ```sh
  node_exporter --collector.disable-defaults --collector.meminfo --collector.filesystem --collector.cpufreq --collector.uname --collector.time --collector.textfile ...
  # + textfile.sh cron */1: battery (termux-battery-status), load, cpu busy, thermal -> textfile dir
  ```

- web: prom./alerts./grafana. through haproxy on the wildcard cert

  ```sh
  # prometheus + alertmanager have no login. auth at haproxy, not in the apps: their own --web.config.file basic auth
  # breaks every loopback client (self-scrape, prometheus->alertmanager push, grafana datasource)
  # termux haproxy: no crypt(3) -> userlist must be insecure-password -> keep it out of the repo
  ~/haproxy.d/05-auth.cfg   # userlist monitoring / user mrk insecure-password <pw>, 0600
  backend prom
      http-request auth realm prometheus unless { http_auth(monitoring) }
  # --web.external-url=https://prom.<zone>:8443/ so ntfy alert links point at the public ui
  # 401 with empty body = cheapest l7 answer, no anubis needed in front of auth'd backends
  # grafana: pkg install grafana (12.x). grafana server --homepath $PREFIX/share/grafana --config <ini>
  #   ini [paths] data/logs/plugins/provisioning absolute, [server] http_addr 127.0.0.1 root_url https://grafana.<zone>:8443/
  #   admin pw via GF_SECURITY_ADMIN_PASSWORD in an env file. datasource provisioned from the monitoring repo,
  #   dashboards from a clone of `sandravwc/grafana-dashboards` (file provider re-reads every 10s, read-only in the ui: edit json, push, pull)
  # haproxy metrics: termux build has no PROMEX -> haproxy_exporter on the stats socket (`stats socket ~/haproxy.sock mode 600 level operator`)
  ```

- grafana dashboard json, things that bit

  ```sh
  # stat panel 0/1 -> UP/DOWN: fieldConfig.defaults.mappings [{type: value, options: {"0": {text: DOWN, color: red}, "1": {text: UP, color: green}}}], graphMode none
  # stat text size: options.text {titleSize, valueSize}; two units in one stat = second query + fieldConfig.overrides byName (unit, thresholds)
  # instant: true on stat queries, else stale series from the time range keep showing as tiles
  # rows: {type: row, gridPos h:1}; panels below need y > row y. grouping by service, not by metric type
  # matching panels by title: rows and stats can share a title, filter on type too (deleted the rows once)
  # blackbox probes count as real requests in haproxy counters -> public probes every 2m, not 30s
  ```

- gotchas

  ```sh
  # amtool alert add ... --annotation=summary='"quote the value"'   (new matcher parser)
  # termux-battery-status current: negative = charging (xiaomi). "plugged + current > 0 for 10m" = the cable alert
  ```
