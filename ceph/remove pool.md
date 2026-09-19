---
title: ceph/remove pool
tags: [ceph]
---

- assuming proxmox hv [[hypervisors/proxmox]]
- assuming consistent hostnames and monitor names
  ```sh
  # on proxmox, config is synchronized using corosync, hence edit only on master node
  sed -i "s#mon_allow_pool_delete = false#mon_allow_pool_delete = true#g" /etc/ceph/ceph.conf
  ceph config set mon mon_allow_pool_delete true
  
  # assuming 5 monitors
  printf "%s\n" {1..5} |xargs -I % ceph config get mon.proxmox0% mon_allow_pool_delet
  # restart monitors
  for a in {1..5}; do ssh -n proxmox0"${a}".intern.example.de 'systemctl restart ceph-mon@proxmox0"${a}"; sleep 5'
  # check if config is really set
  for a in {1..5}; do ssh -n proxmox0"${a}".intern.example.de 'ceph --admin-daemon /var/run/ceph/ceph-mon.proxmox0"${a}".asok config show | grep mon_allow_pool_delete'
  printf "%s\n" {1..5} |xargs -I % ceph config get mon.proxmox0% mon_allow_pool_delete
  
  # delet
  ceph osd pool delete k8s_stretch_dev k8s_stretch_dev --yes-i-really-really-mean-it
  ```
  - revert `mon_allow_pool_delete = false` after pool deleted for enhanced labour procurement measure reasons
    ```sh
    sed -i "s#mon_allow_pool_delete = true#mon_allow_pool_delete = false#g" /etc/ceph/ceph.conf
    ceph config set mon mon_allow_pool_delete false
    for a in {1..5}; do ssh -n proxmox0"${a}".intern.example.de 'systemctl restart ceph-mon@proxmox0"${a}"; sleep 5'
    ```
