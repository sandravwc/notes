---
title: cheat sheet/lvm
---

- create fs + mount
  ```sh
  pvcreate /dev/vdb
  vgcreate abcdef /dev/vdb
  lvcreate -n data -l 100%FREE abcdef
  mkfs.ext4 /dev/mapper/abcdef-data
  
  blkid
  /dev/mapper/abcdef-data: UUID="f4458d15-a4bb-4a8d-948c-08bc32ce7ab5" TYPE="ext4"
  
  echo "UUID=f4458d15-a4bb-4a8d-948c-08bc32ce7ab5 /abcdef-data		ext4	defaults	1 2" >> /etc/fstab
  ```
- extend fs
  ```sh
  pvresize /dev/vdb
  lvextend -r -l +100%FREE /dev/mapper/abcdef-data
  ```
