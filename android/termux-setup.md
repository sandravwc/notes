---
title: android/termux-setup
---

- comfy termux base (native, no proot)

  - ```sh
    pkg install -y neovim bash-completion bat fzf git make gawk fastfetch screen
    git clone https://github.com/akinomyoga/ble.sh.git ~/workdir/ble.sh
    cd ~/workdir/ble.sh && make install INSDIR="$HOME/.local/lib/blesh"
    ```

    full script + .bashrc/.blerc/init.vim/.screenrc: [comfy_script.sh](https://github.com/sandravwc/termux-setup/blob/master/comfy_script.sh)
    no atuin -- no arm/termux build exists, stripped from the script
- bionic locale + $USER (separate, applied after the base script)

  - ```sh
    # bionic only really backs C.UTF-8, not en_US.UTF-8 -- ble.sh warns "locale C broken" regardless of LANG/LC_ALL
    echo 'export LANG=C.UTF-8' >> ~/.bashrc
    echo 'export LC_ALL=C.UTF-8' >> ~/.bashrc
    echo 'export USER=$(id -un)' >> ~/.bashrc
    ```

  - ```sh
    # .blerc -- suppress the leftover (cosmetic, unfixable without patching bionic) warning
    function ble/util/notify-broken-locale { :; }
    ```

- ssh (separate from the comfy script)

  - ```sh
    pkg install openssh termux-services
    sshd            # port 8022 always -- no root, can't bind <1024
    sv-enable sshd  # needs a termux restart first so runsvdir initializes
    # sv needs $SVDIR set (sourced via .bashrc) -- non-interactive ssh exec needs it passed explicitly
    ```

    mesh config: [ssh_config](https://github.com/sandravwc/termux-setup/blob/master/ssh_config)
- termux:boot -- wake lock

  - ```sh
    pkg install termux-api
    mkdir -p ~/.termux/boot
    ```

    script: [termux-boot-wake-lock.sh](https://github.com/sandravwc/termux-setup/blob/master/termux-boot-wake-lock.sh) -- needs both Termux:Boot + Termux:API apps installed (same build source as Termux itself)
- google play termux is dead, don't mix sources

  - ```txt
    play store termux build is frozen/deprecated upstream -- termux-api's KeepAliveService silently fails to resolve against it even with a matching termux-api app version
    same termux-api app version works fine paired with a github/f-droid termux build
    fix: uninstall play termux, install github release apk (same source as termux-api/termux-boot), reinstall
    adb install fails with INSTALL_FAILED_VERIFICATION_FAILURE on a fresh device -> disable play protect verifier first (see android/hyperos-adb-install)
    ```
