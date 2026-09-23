---
title: android/no-root-mount-limits
---

- no fuse/nfs/cifs mount without root, confirmed

    ```sh
    ls -la /dev/fuse                          # crw------- root root, denied even to adb shell (uid 2000)
    mount -t nfs ...                          # inside proot: exit 0 but silent no-op, proot fakes mount success it can't implement
    unshare --user --map-root-user id         # "Invalid argument" -- unprivileged userns creation locked down
    ```

  - SAF-based apps (CIFS Documents Provider, SFTP-SAF...) only expose content:// to the picker, not a real path a native process can open()
  - fallback: periodic rsync/rclone sync, or attach the drive directly / to something that can actually mount it
  - other direction works: termux can *export* (sftp, [[android/nfs-export]])
