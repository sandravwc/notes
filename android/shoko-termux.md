---
title: android/shoko-termux
---

- shoko server native in termux on the poco, repo `sandravwc/shoko-termux` (readme = install)
  - `~/shoko/{app,dotnet,repo,libifaddrs_shim.so}`, data `~/.shoko/Shoko.CLI`, sv `shoko`, `anubis-shoko`, `nfs`
  - via [[android/glibc-runner]], ssd export [[android/nfs-export]], public via [[android/haproxy-termux]] + [[android/anubis-termux]]

- migrate data out of proot

  ```sh
  cp -a $PREFIX/var/lib/proot-distro/containers/debian/rootfs/home/<user>/.shoko ~/.shoko
  sed -i "s|/home/<user>/|$HOME/|g" ~/.shoko/Shoko.CLI/settings-server.json
  pkg install sqlite
  sqlite3 ~/.shoko/Shoko.CLI/SQLite/JMMServer.db3 "update ImportFolder set ImportFolderLocation='$HOME/...' where ImportFolderID=2"
  ```

- gotchas

  ```sh
  # log "Checking LAN Connectivity…" and nothing after = zero usable ifaces (NoInterfaces branch has no log line) -> shim missing, anidb stays off
  # sv restart shoko under proot never worked (TERM lost in ptrace), native it does
  # pkill -f Shoko.CLI from ssh kills the ssh session too (matches own cmdline). pkill -x
  # ~/shoko/dotnet: rm -rf sdk packs templates after install, 700M -> 200M, runtime = host/ + shared/
  ```
