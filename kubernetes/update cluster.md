---
title: kubernetes/update cluster
---

## - you need to update from version to version since kubernetes doesnt support skipping major versions

- there is a separate page with important information from Kubernetes for each major version
  - <https://v1-31.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade>
- read release notes on github (if you have nothing else better to do)
  - <https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.32.md>
- procedure:
  - create backup of nodes (LOL)
  - study release notes (LOL)
  - install & run pluto
  - update helm
  - update k8s
    - update k8s repo
    - update k8s binaries
    - run kubeadm upgrade
      - `kubeadm upgrade plan` (optional)
      - on first control plane node only
      - everywhere else
    - dnf upgrade (update os and crio)
      - update crio repo
      - run dnf upgrade
    - reboot
  - repeat for every node
- install pluto

  ```bash
  plutoLatestLinuxAmd64Release="$(curl -s https://api.github.com/repos/FairwindsOps/pluto/releases/latest | jq -r '.assets[] | select(.name | contains("linux_amd64") and endswith(".tar.gz")) | .name')"
  plutoLatestLinuxAmd64ReleaseUrl="$(curl -s https://api.github.com/repos/FairwindsOps/pluto/releases/latest | jq -r '.assets[] | select(.name | contains ("linux_amd64")and endswith(".tar.gz")) | .browser_download_url')"
  wget --quiet "${plutoLatestLinuxAmd64ReleaseUrl}"
  tar -xzf "${plutoLatestLinuxAmd64Release}"
  mv pluto /root/bin/pluto
  ```

- run pluto

  ```bash
  pluto detect-helm
  NAME                              KIND      VERSION                     REPLACEMENT            REMOVED   DEPRECATED   REPL AVAIL
  iljuchin-jenkins-test/jenkins     Ingress   networking.k8s.io/v1beta1   networking.k8s.io/v1   true      true         true
  production-jenkins-test/jenkins   Ingress   networking.k8s.io/v1beta1   networking.k8s.io/v1   true      true         true
  ```

- update helm

  ```bash
  helmLatestVersion=$(curl -s https://api.github.com/repos/helm/helm/releases/latest | jq -r '.tag_name')
  wget --quiet https://get.helm.sh/helm-"${helmLatestVersion}"-linux-amd64.tar.gz
  tar xzf helm-"${helmLatestVersion}"-linux-amd64.tar.gz
  mv linux-amd64/helm /usr/local/bin/helm
  rm -rf linux-amd64/
  ```

- update k8s
  - get k8s release
    - `kubernetesStableRelease=$(curl -s https://endoflife.date/api/kubernetes.json | jq -r '.[0].cycle')`
    - `kubernetesLatestRelease=$(curl -s https://endoflife.date/api/kubernetes.json | jq -r '.[0].latest')`
  - update k8s repo

    ```bash
    cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://pkgs.k8s.io/core:/stable:/v${kubernetesStableRelease}/rpm/
    enabled=1
    gpgcheck=1
    gpgkey=https://pkgs.k8s.io/core:/stable:/v${kubernetesStableRelease}/rpm/repodata/repomd.xml.key
    exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
    EOF
    ```

  - alternatively, just search and replace your repofiles if you're in the loop

    ```bash
    sed -i 's/v1.33/v1.34/g' /etc/yum.repos.d/{kubernetes,cri-o}.repo
    ```

  - drain node and check if pods are running there

    ```bash
    kubectl drain \
    --force \
    --ignore-daemonsets \
    --grace-period=60 \
    --timeout=60s \
    --delete-emptydir-data node_name
    
    kubectl get pods \
    --all-namespaces \
    --field-selector spec.nodeName=node_name
    ```

  - update k8s binaries

    ```bash
    dnf update --assumeyes --disableexcludes=kubernetes \
    kubeadm \
    kubectl \
    kubelet
    ```

  - run kubeadm upgrade
    - `kubeadm upgrade plan` (optional)

      ```sh
      Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
      COMPONENT   NODE                                       CURRENT   TARGET
      kubelet     k8s-node01.example.internal          v1.30.3   v1.30.11
      kubelet     k8s-node02.example.internal          v1.30.3   v1.30.11
      kubelet     k8s-node03.example.internal          v1.30.3   v1.30.11
      kubelet     k8s-node04.example.internal          v1.30.3   v1.30.11
      kubelet     k8s-node05.example.internal          v1.30.3   v1.30.11
      kubelet     k8s-node06.example.internal          v1.30.3   v1.30.11
      kubelet     k8s-master01.example.internal   v1.30.3   v1.30.11
      
      Upgrade to the latest version in the v1.30 series:
      
      COMPONENT                 NODE                                       CURRENT    TARGET
      kube-apiserver            k8s-node02.example.internal          v1.30.3    v1.30.11
      kube-apiserver            k8s-node03.example.internal          v1.30.3    v1.30.11
      kube-apiserver            k8s-master01.example.internal   v1.30.3    v1.30.11
      kube-controller-manager   k8s-node02.example.internal          v1.30.3    v1.30.11
      kube-controller-manager   k8s-node03.example.internal          v1.30.3    v1.30.11
      kube-controller-manager   k8s-master01.example.internal   v1.30.3    v1.30.11
      kube-scheduler            k8s-node02.example.internal          v1.30.3    v1.30.11
      kube-scheduler            k8s-node03.example.internal          v1.30.3    v1.30.11
      kube-scheduler            k8s-master01.example.internal   v1.30.3    v1.30.11
      kube-proxy                                                           1.30.3     v1.30.11
      CoreDNS                                                              v1.11.1    v1.11.1
      etcd                      k8s-node02.example.internal          3.5.12-0   3.5.12-0
      etcd                      k8s-node03.example.internal          3.5.12-0   3.5.12-0
      etcd                      k8s-master01.example.internal   3.5.12-0   3.5.12-0
      
      You can now apply the upgrade by executing the following command:
      
       kubeadm upgrade apply v1.30.11
      
      Note: Before you can perform this upgrade, you have to update kubeadm to v1.30.11.
      ```

    - on first control plane only

      ```sh
      kubeadm upgrade apply v"${kubernetesLatestRelease}"
      systemctl daemon-reload
      systemctl restart kubelet
      ```

    - everywhere else

      ```sh
      kubeadm upgrade node
      systemctl daemon-reload
      systemctl restart kubelet
      ```

- dnf upgrade (update os and crio)
  - update crio repo

    ```bash
    cat <<EOF | tee /etc/yum.repos.d/cri-o.repo
    [cri-o]
    name=CRI-O
    baseurl=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/v${kubernetesStableRelease}/rpm/
    enabled=1
    gpgcheck=1
    gpgkey=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/v${kubernetesStableRelease}/rpm/repodata/repomd.xml.key
    EOF
    ```

  - run dnf upgrade

    ```bash
    dnf upgrade --assumeyes
    reboot
    ```

- uncordon after reboot done

  ```bash
  kubectl uncordon NODE
  ```

***

- ## can be done programmaticaly with a simple script

  ```bash
  #!/usr/bin/env bash
  nodeType="${1}"

  if ! dnf list installed jq &> /dev/null
  then
    exit 2
  fi

  if [[ ! "${nodeType}" =~ ^(master|worker)$ ]]
  then
    printf "%s\n%s\n" "you need to provide nodeType ${0} master|worker" "depending on nodeType, kubeadm upgrade steps will differ"
    exit 2
  fi

  kubernetesStableRelease=$(curl -s https://endoflife.date/api/kubernetes.json | jq -r '.[0].cycle')
  kubernetesLatestRelease=$(curl -s https://endoflife.date/api/kubernetes.json | jq -r '.[0].latest')

  cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
  [kubernetes]
  name=Kubernetes
  baseurl=https://pkgs.k8s.io/core:/stable:/v${kubernetesStableRelease}/rpm/
  enabled=1
  gpgcheck=1
  gpgkey=https://pkgs.k8s.io/core:/stable:/v${kubernetesStableRelease}/rpm/repodata/repomd.xml.key
  exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
  EOF



  cat <<EOF | tee /etc/yum.repos.d/cri-o.repo
  [cri-o]
  name=CRI-O
  baseurl=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/v${kubernetesStableRelease}/rpm/
  enabled=1
  gpgcheck=1
  gpgkey=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/v${kubernetesStableRelease}/rpm/repodata/repomd.xml.key
  exclude=cri-o
  EOF

  dnf update --assumeyes --disableexcludes=kubernetes,cri-o \
    kubeadm \
    kubectl \
    kubelet \
    cri-o

  if [[ "${nodeType}" == master ]]
  then
    echo "running checks for kubeadm upgrade apply v${kubernetesLatestRelease}"
    kubeadm upgrade plan
    echo "RUN UPDATE AS PRESCRIBED"
    echo "restart kubelet afterwards!!"
    echo "run systemctl daemon-reload"
    echo "run systemctl restart kubelet"
  elif [[ "${nodeType}" == worker ]]
  then
    kubeadm upgrade node
    systemctl daemon-reload
    systemctl restart kubelet
  fi
  ```
