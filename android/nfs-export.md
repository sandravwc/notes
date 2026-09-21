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
  # fstab. soft, not hard (see below)
  <phone ip>:/ /mnt/shoko_ds1 nfs port=2049,mountport=2049,tcp,nolock,vers=3,soft,timeo=50,retrans=3,nofail,_netdev,x-systemd.automount 0 0
  # ~70 MB/s write, ~45 read over wifi (poco f5 pro, usb ssd). df on the mount lies, rclone vfs
  ```

- stale handles after a rename through the mount

  ```sh
  # rclone handles are path-based (disk cache = handle<->path). mv a dir via the client -> every handle for that tree ESTALE
  # kernel nfsd keys on inodes, rclone can't. restarting rclone changes nothing
  # hard mount: client retries forever, gnome/ls/shell wedge, reboot. soft: EIO after timeo*retrans
  # move big trees on the poco (ssh mv, or shoko's own rename), not through the mount
  # getting data onto the ssd: rsync over ssh, not the nfs mount. no handles, resumable
  rsync -avP --remove-source-files src/ poco:/storage/XXXX-XXXX/Anime/ && find src -type d -empty -delete
  # never `ls` a suspect nfs mount from a session you need: cat /proc/mounts, dmesg | grep -i nfs, timeout 3 stat
  # dmesg "[UFW BLOCK] SRC=<poco> SPT=2049" after a reboot = server still talking to the dead tcp session, harmless
  ```

- stale handles after a rename through the mount

  ```sh
  # rclone handles are path-based (disk cache = handle<->path). mv a dir via the client -> every handle for that tree ESTALE
  # kernel nfsd keys on inodes, rclone can't. restarting rclone changes nothing
  # hard mount: client retries forever, gnome/ls/shell wedge, reboot. soft: EIO after timeo*retrans
  # move big trees on the poco (ssh mv, or shoko's own rename), not through the mount
  # getting data onto the ssd: rsync over ssh, not the nfs mount. no handles, resumable
  rsync -avP --remove-source-files src/ poco:/storage/XXXX-XXXX/Anime/ && find src -type d -empty -delete
  # never `ls` a suspect nfs mount from a session you need: cat /proc/mounts, dmesg | grep -i nfs, timeout 3 stat
  # dmesg "[UFW BLOCK] SRC=<poco> SPT=2049" after a reboot = server still talking to the dead tcp session, harmless
  ```

- "server not responding, timed out" while shoko imports

  ```sh
  # not stale. shoko hashing = every file read through android's fuse daemon at full speed, rclone's requests queue behind it,
  # soft mount gives up after timeo*retrans. passes when the hash run ends. dmesg tells the two apart: "Stale file handle" vs "not responding"
  # check: shoko log for Hasher / ProcessFileJob, termux_load1 on the poco
  ```

- runit: [[android/shoko-termux]] `deploy/sv-nfs.run`
- sshfs alternative: nothing to install, sshd sftp subsystem is on by default, slower (ssh crypto on the phone)
