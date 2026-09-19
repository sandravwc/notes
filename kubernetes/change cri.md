---
title: kubernetes/change cri
tags: [kubernetes]
---

- from cri-o to containerd.io
  ```sh
  dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
  kubectl cordon k8s-master01.example.internal
  dnf remove podman
  dnf remove cri-o -y
  dnf install containerd.io -y
  mkdir -p /etc/containerd
  containerd config default | tee /etc/containerd/config.toml
  kubectl edit no k8s-master01.example.internal
       kubeadm.alpha.kubernetes.io/cri-socket: unix:///run/containerd/containerd.sock
  Edit the file `/var/lib/kubelet/kubeadm-flags.env` and add the containerd runtime to the flags;
  `--container-runtime-endpoint=unix:///run/containerd/containerd.sock`
    -> deprecated btw
  systemctl enable containerd --now
  
  reboot and uncordon
  ```
