---
title: android/proot-debian
---

- debian userland in termux via proot-distro

  - ```sh
    pkg install proot-distro
    proot-distro install debian
    proot-distro login debian    # real glibc, locale-gen works normally
    ```

- gotchas

  - ```sh
    # proot-distro login = login shell (bash -l) -> reads .bash_profile not .bashrc; source .bashrc from .bash_profile
    # `--user` + backgrounding = process dies when the login command exits (proot --kill-on-exit) -- su - <user> -c '<cmd>' after logging in as root instead
    # screen daemonizing breaks *inside* proot (ptrace loses the double-fork) -- run screen outside proot, wrapping the login command as its window:
    screen -dmS debian proot-distro login debian --user srv-admin
    # screen socket dir root-owned/700 inside proot rootfs -> export SCREENDIR=$HOME/.screen
    # never auto-chain into proot from .bashrc (exec proot-distro login ...) -- pty output can vanish, hangs every future termux launch
    # host dirs: --bind /storage/XXXX-XXXX:/mnt/ssd (not --mount, no such flag). per login! sshd/screen/service each need their own --bind
    # first --bind leaves rootfs/mnt/ssd as a mode-000 placeholder -> a login without the bind shows "Permission denied", not "not found"
    # sv restart of a dotnet app inside proot: TERM never arrives, kill -9 the proot pid
    proot-distro remove debian    # "container busy (PID n: login)" -> kill that pid first
    ```

- replaced by [[android/glibc-runner]] for glibc binaries, 2026-09-19
