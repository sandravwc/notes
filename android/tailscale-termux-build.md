---
title: android/tailscale-termux-build
---

- official static binary dies on termux

  ```txt
  netmon.New: route ip+net: netlinkrib: permission denied
  android 11+ denies netlink to app uids. fix is in termux's go stdlib patch (net.Interfaces falls back on EPERM), so build with termux go
  ```

- build

  ```sh
  pkg install golang
  git clone --depth 1 --branch v1.102.4 https://github.com/tailscale/tailscale ~/tailscale-src && cd ~/tailscale-src
  # acme/cert code is excluded on android, drop the tag only inside //go:build lines
  for f in $(grep -rlE '^//go:build.*android' feature/acme feature/condregister ipn/localapi cmd/tailscale/cli); do
    sed -i -E '/^\/\/go:build/{s/!android && //; s/ && !android//; s/ \|\| android//}' "$f"; done
  TAGS=ts_omit_ssh,ts_omit_tap,ts_omit_bird,ts_omit_capture,ts_omit_clientupdate,ts_omit_drive,ts_omit_taildrop,ts_omit_doctor,ts_omit_kube,ts_omit_aws,ts_omit_oauthkey,ts_omit_identityfederation,ts_omit_posture,ts_omit_syspolicy,ts_omit_wakeonlan,ts_omit_tailnetlock,ts_omit_relayserver,ts_omit_netlog,ts_omit_portlist,ts_omit_linkspeed,ts_omit_linuxdnsfight,ts_omit_captiveportal,ts_omit_ace,ts_omit_appconnectors,ts_omit_debugportmapper,ts_omit_portmapper,ts_omit_remoteconfig,ts_omit_routecheck,ts_omit_runtimemetrics,ts_omit_sdnotify,ts_omit_serviceclientprefs,ts_omit_tundevstats,ts_omit_useproxy,ts_omit_systray,ts_omit_webclient,ts_omit_desktop_sessions,ts_omit_tpm,ts_omit_c2n,ts_omit_conn
  go list -tags "$TAGS" ./cmd/tailscaled ./cmd/tailscale   # constraint check in seconds, before the compile
  go build -p 3 -trimpath -tags "$TAGS" -ldflags="-s -w" -o ~/.tailscale/ ./cmd/tailscaled ./cmd/tailscale   # ~1 min, -p 3 keeps ram sane
  ~/.tailscale/tailscaled --tun=userspace-networking --statedir=$HOME/.tailscale/state --socket=$HOME/.tailscale/sock --port=41641
  ~/.tailscale/tailscale --socket=$HOME/.tailscale/sock up
  # sed on the whole file (not only build lines) corrupts c2n_test.go etc: "parsing //go:build line: unexpected token !"
  # dropped for mealprep: every client needs the tailscale app. own domain + acme dns-01 instead, see [[cheat sheet/acme-autodns]]
  ```
