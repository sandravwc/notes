---
title: kubernetes/install cluster
---

- copypaste
  
  ```sh
  # install docker-ce:
  yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
  dnf install docker-ce
  
  # add kubernetes repo:
  # /etc/yum.repos.d/kubernetes.repo
  [kubernetes]
  name=Kubernetes
  baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
  enabled=1
  gpgcheck=1
  repo_gpgcheck=1
  gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
  exclude=kubelet kubeadm kubectl
  
  
  # install kubernetes:
  cat /etc/yum.repos.d/kubernetes.repo
  dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
  # add override config to docker and enable service starting it:
  cat /etc/systemd/system/docker.service.d/override.conf
  [Service]
  Environment="DOCKER_ARGS=--exec-opt native.cgroupdriver=systemd"
  ExecStart=
  ExecStart=/usr/bin/dockerd $DOCKER_ARGS
  systemctl daemon-reload
  systemctl enable docker --now
  
  #init kubernetes cluster:
  kubeadm init --control-plane-endpoint "k8s-master-proceed-integration:6443" --pod-network-cidr=192.168.0.0/16 --upload-certs
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  kubectl get nodes
  
  # install calico network:
  wget https://docs.projectcalico.org/manifests/tigera-operator.yaml
  kubectl create -f tigera-operator.yaml 
  wget https://docs.projectcalico.org/manifests/custom-resources.yaml
  kubectl create -f custom-resources.yaml
  kubectl taint nodes --all node-role.kubernetes.io/master
  ```
