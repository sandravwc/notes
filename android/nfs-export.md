---
title: android/nfs-export
---

- export a dir from termux over nfs, no root, no kernel nfsd

  ```sh
  pkg install rclone
  rclone serve nfs /storage/XXXX-XXXX --addr <lan ip>:2049 --nfs-cache-type disk --vfs-cache-mode writes --umask 000
  # nfsv3, no auth -> lan bind only
  # --nfs-cache-type disk: handles survive restart, no stale mounts on clients
  # /storage/XXXX-XXXX = usb drive, android fuse, rw for termux after termux-setup-storage
  ```

- client, port must be given (no portmapper)

  ```sh
  # fstab
  <phone ip>:/ /mnt/shoko_ds1 nfs port=2049,mountport=2049,tcp,nolock,vers=3,nofail,_netdev,x-systemd.automount 0 0
  # ~70 MB/s write, ~45 read over wifi (poco f5 pro, usb ssd). df on the mount lies, rclone vfs
  ```

- runit: [[android/shoko-termux]] `deploy/sv-nfs.run`
- sshfs alternative: nothing to install, sshd sftp subsystem is on by default, slower (ssh crypto on the phone)
