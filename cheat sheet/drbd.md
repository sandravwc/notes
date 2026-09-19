---
title: cheat sheet/drbd
---

- resize disk
  ```sh
  resize block device first
  assuming lvm with this setup:
  
  [00:42:08][root@nfs02:~]$ drbdadm status
  nfs role:Primary
    disk:UpToDate
    nfs01.intern.example.de role:Secondary
      peer-disk:UpToDate
  
  [00:42:13][root@nfs02:~]$ lsblk
  NAME       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
  sr0         11:0    1 1024M  0 rom
  xvda       202:0    0   32G  0 disk
  |-xvda1    202:1    0    1G  0 part /boot
  |-xvda2    202:2    0    8G  0 part [SWAP]
  `-xvda3    202:3    0   23G  0 part /
  xvdb       202:16   0  256G  0 disk
  `-nfs-data 253:0    0  256G  0 lvm
    `-drbd0  147:0    0  192G  0 disk /exports
  
  
  on every node:
  pvresize /dev/xvdb
  lvextend -l +100%FREE /dev/mapper/nfs-data
  
  then on master:
  drbdadm resize nfs
  resize2fs /dev/drbd0
  
  and then just wait for all nodes to catch up
  ```
