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
  # TimeZoneNotFoundException 'Tokyo Standard Time': .net reads /usr/share/zoneinfo, termux has no tzdata pkg at all
  rsync -a --exclude right --exclude posix /usr/share/zoneinfo/ poco:shoko/zoneinfo/   # TZDIR=$HOME/shoko/zoneinfo in run
  # "failed to read MediaInfo" on every file = shoko spawns termux's bionic `mediainfo`, child inherited the glibc LD_PRELOAD shim
  patchelf --add-needed ~/shoko/libifaddrs_shim.so ~/shoko/app/Shoko.CLI   # DT_NEEDED instead, no env, children clean. shoko stopped or "Text file busy"
  # sv restart: shoko drains jobs 30s+, sv says "timeout" at 7s, it does stop. sv kill when in a hurry
  # log "Unable to load shared library 'librhash'" -> C# fallback hashing, slow + hot. pkg install rhash-glibc; ln -s $PREFIX/glibc/lib/librhash.so ~/shoko/app/
  ```

- a network timeout that is actually a ban

  ```txt
  before blaming the build, proot or the network stack: send a raw credential-free
  ping to the api first. some apis answer a bare ping with an explicit ban notice
  while silently dropping authenticated requests, which looks like a plain timeout.
  stop retrying once suspected, it extends the ban.
  ```
