---
title: hypervisors/xcp-ng
---

- change disk size using xe

  ```sh
  [root@xcp-ng-node01 ~]# xe vm-list name-label=db.petershop.de
  uuid ( RO)           : f272f60e-a9ac-b97d-a7af-f7936167cf28
     name-label ( RW): db.petershop.de
    power-state ( RO): running
  [root@xcp-ng-node01 ~]# xe vm-disk-list vm=db.petershop.de
  Disk 0 VBD:
  uuid ( RO)             : b42cb713-da01-01d1-b7bf-94cca5c80ab8
    vm-name-label ( RO): db.petershop.de
       userdevice ( RW): 0
  Disk 0 VDI:
  uuid ( RO)             : 9a4ec281-f9a2-4632-a9ed-a5c8da7140c7
       name-label ( RW): db.petershop.de 0
    sr-name-label ( RO): Local RAID on xcp-ng-node01
     virtual-size ( RO): 295279001600
  
  # echo $((295279001600/2**30)) = 275(GiB)
  
  [root@xcp-ng-node01 ~]# xe vm-shutdown uuid=f272f60e-a9ac-b97d-a7af-f7936167cf28
  
  [root@xcp-ng-node01 ~]# xe vdi-resize uuid=9a4ec281-f9a2-4632-a9ed-a5c8da7140c7 virtual-size=429496729600
  # 429496729600 = echo $((400*2**30))
  
  xe vm-start uuid=f272f60e-a9ac-b97d-a7af-f7936167cf28
  ```

- create storage repositry

  ```sh
  [15:44 xcp-ng03 ~]# xe sr-create content-type=user device-config:device=/dev/disk/by-id/scsi-36c81f660ca0e3200298680060b87e542 host-uuid=051175b4-f335-441a-a118-ad12caae79c1 name-label="SSD" shared=false type=ext
  ```

- expand storage repositry file system without downtime

  ```sh
  #manchmal wollen leute halt keine downtimes haben; mit manchen raid controllern kann man dinge machen wie zum beispiel online raid expansion.
  #progress einsehbar mit megacli dann
  
  MegaCli -LDInfo -L0 -a0
  
  #naja okay wenn das dann fertig ist guckt man dumm weil lsblk sagt immer noch ne ich bin $size_b4_resize groß
  #dann kann man sagen ja okay dann schau halt nochmal nach weil das letzte mal als ich den controller gefragt habe, hat der gesagt $size_after_resize, und das geht so:
  
  echo "1" > /sys/class/block/sda/device/rescan 
  
  #dann muss man nurnoch (wenn man gpt hat) sagen, dass $sachen am ende der disk geschrieben werden sollen
  
  gdisk /dev/sda
  
  Command (? for help): x
  Expert command (? for help): e
  Relocating backup data structures to the end of the disk
  Expert command (? for help): w
  Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
  PARTITIONS!!
  
  Do you want to proceed? (Y/N): y
  
  #dann geht man nochmal rein und tut entweder das vorhandene pv erweitern, oder aber (das ist cooler) man erstellt neue part und schiebt das dann ins vg
  
  [11:13 xcp-ng02 ~]# gdisk /dev/sda
  GPT fdisk (gdisk) version 0.8.6
  
  Partition table scan:
    MBR: protective
    BSD: not present
    APM: not present
    GPT: present
  
  Found valid GPT with protective MBR; using GPT.
  
  Command (? for help): n
  Partition number (7-128, default 7): 
  First sector (34-7497318366, default = 3748659200) or {+-}size{KMGTP}: 3748659200
  Last sector (3748659200-7497318366, default = 7497318366) or {+-}size{KMGTP}: 7497318366
  Current type is 'Linux filesystem'
  Hex code or GUID (L to show codes, Enter = 8300): 8E00
  Changed type of partition to 'Linux LVM'
  
  Command (? for help): w
  
  Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
  PARTITIONS!!
  
  Do you want to proceed? (Y/N): y
  OK; writing new GUID partition table (GPT) to /dev/sda.
  Warning: The kernel is still using the old partition table.
  The new table will be used at the next reboot.
  
  # dann jodelt man sich noch bisschen durchs lvm, kennt man ja
  
  [11:14 xcp-ng02 ~]# partprobe
  [11:14 xcp-ng02 ~]# pvcreate /dev/sda7
    Physical volume "/dev/sda7" successfully created.
  [11:18 xcp-ng02 ~]# vgs
    VG                                              #PV #LV #SN Attr   VSize  VFree
    XSLocalEXT-29e3c93c-91aa-82ec-f723-f13a9a9f0142   1   1   0 wz--n- <1,71t    0 
  [11:18 xcp-ng02 ~]# vgextend XSLocalEXT-29e3c93c-91aa-82ec-f723-f13a9a9f0142 /dev/sda7
    Volume group "XSLocalEXT-29e3c93c-91aa-82ec-f723-f13a9a9f0142" successfully extended   29e3c93c-91aa-82ec-f723-f13a9a9f0142 XSLocalEXT-29e3c93c-91aa-82ec-f723-f13a9a9f0142 -wi-ao---- <1,71t                                                    
  [11:19 xcp-ng02 ~]# df -hT
  Dateisystem                                                                                               Typ      Größe Benutzt Verf. Verw% Eingehängt auf
  devtmpfs                                                                                                  devtmpfs  3,6G     56K  3,6G    1% /dev
  tmpfs                                                                                                     tmpfs     3,6G    516K  3,6G    1% /dev/shm
  tmpfs                                                                                                     tmpfs     3,6G     11M  3,6G    1% /run
  tmpfs                                                                                                     tmpfs     3,6G       0  3,6G    0% /sys/fs/cgroup
  /dev/sda1                                                                                                 ext3       18G    2,2G   15G   14% /
  xenstore                                                                                                  tmpfs     3,6G       0  3,6G    0% /var/lib/xenstored
  /dev/sda5                                                                                                 ext3      3,9G    203M  3,5G    6% /var/log
  /dev/mapper/XSLocalEXT--29e3c93c--91aa--82ec--f723--f13a9a9f0142-29e3c93c--91aa--82ec--f723--f13a9a9f0142 ext3      1,7T    1,3T  377G   77% /run/sr-mount/29e3c93c-91aa-82ec-f723-f13a9a9f0142
  192.168.13.20:/opt/iso                                                                                   nfs       922G    7,5G  915G    1% /run/sr-mount/351efba3-d167-d7d4-b602-7145b9eec1e8
  192.168.13.30:/opt/recovery/7e169b9c-d1c8-933b-01d2-01beda5bcf02                                         nfs        15T     11T  3,7T   75% /run/sr-mount/7e169b9c-d1c8-933b-01d2-01beda5bcf02
  tmpfs                                                                                                     tmpfs     729M       0  729M    0% /run/user/0
  [11:20 xcp-ng02 ~]# lvextend -r -l +100%FREE /dev/mapper/XSLocalEXT--29e3c93c--91aa--82ec--f723--f13a9a9f0142-29e3c93c--91aa--82ec--f723--f13a9a9f0142
    Size of logical volume XSLocalEXT-29e3c93c-91aa-82ec-f723-f13a9a9f0142/29e3c93c-91aa-82ec-f723-f13a9a9f0142 changed from <1,71 TiB (446972 extents) to 3,45 TiB (904571 extents).
    Logical volume XSLocalEXT-29e3c93c-91aa-82ec-f723-f13a9a9f0142/29e3c93c-91aa-82ec-f723-f13a9a9f0142 successfully resized.
  resize2fs 1.42.9 (28-Dec-2013)
  Das Dateisystem auf /dev/mapper/XSLocalEXT--29e3c93c--91aa--82ec--f723--f13a9a9f0142-29e3c93c--91aa--82ec--f723--f13a9a9f0142 ist auf /run/sr-mount/29e3c93c-91aa-82ec-f723-f13a9a9f0142 eingehängt; Online-Grössenveränderung nötig
  old_desc_blocks = 110, new_desc_blocks = 221
  ```
