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
    ```
