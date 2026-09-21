---
title: android/glibc-runner
---

- run glibc linux-arm64 binaries natively in termux, no proot

  ```sh
  pkg install glibc-repo
  pkg install glibc-runner libicu-glibc          # $PREFIX/glibc = full glibc
  grun -c ./binary                               # patches ELF interpreter + rpath once, then runs as itself (/proc/self/exe correct)
  env -u LD_PRELOAD ./binary                     # termux LD_PRELOAD is a bionic .so, glibc loader chokes: "libc.so: invalid ELF header"
  # never export a glibc .so in LD_PRELOAD in a normal shell -> every bionic program after it fails to link
  # same for the app's children: a glibc app that spawns bionic tools (mediainfo, ffmpeg) must not carry LD_PRELOAD. patchelf --add-needed the shim into the exe instead
  # gcc: pkg install gcc-glibc binutils-glibc; env -u LD_PRELOAD PATH=$PREFIX/glibc/bin:$PATH gcc ...
  ```

- .net 8 this way

  ```sh
  curl -sSL https://dot.net/v1/dotnet-install.sh | bash -s -- --runtime aspnetcore --channel 8.0 --arch arm64 --install-dir ~/app/dotnet
  grun -c ~/app/dotnet/dotnet; grun -c ~/app/App    # apphost + muxer
  DOTNET_ROOT=~/app/dotnet DOTNET_gcServer=0 ./App
  # sdk (dotnet build/restore) dies on a named mutex: /tmp/.dotnet/shm hardcoded, termux has no /tmp. runtime fine, build elsewhere
  # timezones: no tzdata pkg in termux, copy /usr/share/zoneinfo from any linux box, TZDIR=...
  # vs linux-bionic-arm64 RID: official, but any glibc-only native dep (Magick.Native...) blocks it
  ```

- what android selinux denies to apps, glibc side effects

  ```sh
  # netlink bind()            -> getifaddrs() EACCES, python socket.if_nameindex() EACCES, ip addr fails
  # SIOCGIFINDEX on AF_UNIX   -> if_nametoindex() = 0
  # /proc/net/*, /sys/class/net, /proc/sys/net -> EACCES. also /proc/stat, /proc/loadavg, /proc/vmstat, /proc/uptime (see [[android/prometheus-termux]])
  # same ioctls on an AF_INET socket work: SIOCGIFCONF, SIOCGIFFLAGS, SIOCGIFINDEX (SIOCGIFHWADDR not)
  # fix = LD_PRELOAD shim reimplementing getifaddrs/if_nametoindex over ioctl: shoko-termux/shim/ifaddrs_shim.c
  # .net PAL needs an AF_PACKET entry per iface too, it sizes its addr array as (entries - inet entries)
  # proot hides all this: termux proot emulates netlink in userspace, so it "works" there and breaks native
  ```

- was: [[android/dotnet-arm64-proot]], [[android/proot-debian]]. used by [[android/shoko-termux]]
