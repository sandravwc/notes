---
title: android/termux-file-transfer
---

- copy files between two termux installs, no root

  - ```sh
    # termux $HOME is sandboxed, run-as needs a debuggable app -- not available
    tar -czf ~/storage/downloads/transfer.tar.gz -C ~ workdir     # source, inside termux
    adb -s <source-serial> pull /sdcard/Download/transfer.tar.gz
    adb -s <target-serial> push transfer.tar.gz /sdcard/Download/
    tar -xzf ~/storage/downloads/transfer.tar.gz -C ~             # target, inside termux
    # fresh termux, no ~/storage yet:
    adb shell pm grant com.termux android.permission.READ_EXTERNAL_STORAGE
    adb shell pm grant com.termux android.permission.WRITE_EXTERNAL_STORAGE
    termux-setup-storage
    ```
