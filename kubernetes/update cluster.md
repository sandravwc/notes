---
title: kubernetes/update cluster
---

- this documentation is intended to simplify the shift to auto update kubernetes clusters
- you need to update from version to version since kubernetes doesnt support skipping major versions
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

  - Backup Script
    full_backup_k8s.sh

       ```sh
       #!/usr/bin/bash
       
       # --- 1. Setup Environment and Directory ---
       
       BACKUP_DIR="/root/backup_k8s_02_12_2025"
       HOSTNAME=$(hostname -f)
       DATE_TIME=$(date +%Y%m%d-%H%M%S)
       SNAP_NAME="etcd-pre-upgrade-${DATE_TIME}.db"
       LOCAL_PATH="${BACKUP_DIR}/${SNAP_NAME}"
       
       echo "Starting Kubernetes pre-upgrade backup..."
       
       # Create the main backup directory
       mkdir -p "$BACKUP_DIR"
       if [ $? -ne 0 ]; then
           echo "❌ ERROR: Failed to create backup directory: ${BACKUP_DIR}"
           exit 1
       fi
       echo "✅ Directory created: ${BACKUP_DIR}"
       
       # --- 2. Find etcd Pod ---
       
       ETCD_POD=$(kubectl -n kube-system get pod -l component=etcd \
                 -o jsonpath="{.items[?(@.metadata.name==\"etcd-${HOSTNAME}\")].metadata.name}" 2>/dev/null)
       
       if [ -z "$ETCD_POD" ]; then
           echo "❌ ERROR: Failed to find etcd pod for this host: etcd-${HOSTNAME}"
           exit 1
       fi
       
       echo "etcd pod for this host is: ${ETCD_POD}"
       echo "Local path for etcd snapshot: ${LOCAL_PATH}"
       
       # --- 3. Save etcd Snapshot ---
       
       # Perform the etcd snapshot and pipe the binary output directly to the local file
       echo "Saving etcd snapshot..."
       kubectl -n kube-system exec -i "$ETCD_POD" -- \
         /bin/sh -c \
         'ETCDCTL_API=3 etcdctl \
           --endpoints=https://127.0.0.1:2379 \
           --cacert=/etc/kubernetes/pki/etcd/ca.crt \
           --cert=/etc/kubernetes/pki/etcd/server.crt \
           --key=/etc/kubernetes/pki/etcd/server.key \
           snapshot save -' > "$LOCAL_PATH"
       
       if [ $? -ne 0 ]; then
           echo "❌ ERROR: Failed to save etcd snapshot."
           rm -f "$LOCAL_PATH" # Clean up potentially corrupted file
           exit 1
       fi
       
       # Check if the snapshot file is non-empty
       if [ ! -s "$LOCAL_PATH" ]; then
           echo "❌ ERROR: etcd snapshot file is empty or missing after save."
           exit 1
       fi
       echo "✅ etcd snapshot saved to ${LOCAL_PATH}"
       
       # --- 4. Copy Critical Configuration Files ---
       
       echo "Copying critical configuration files and directories..."
       
       # Create a subdirectory for the associated config files (optional but good practice)
       CONFIG_DIR="${BACKUP_DIR}/config_files-${DATE_TIME}"
       mkdir -p "$CONFIG_DIR"
       if [ $? -ne 0 ]; then
           echo "❌ ERROR: Failed to create config directory: ${CONFIG_DIR}"
           exit 1
       fi
       
       # Copy the entire PKI directory
       sudo cp -R /etc/kubernetes/pki "$CONFIG_DIR/pki"
       if [ $? -ne 0 ]; then
           echo "❌ ERROR: Failed to copy PKI directory."
           exit 1
       fi
       echo "✅ PKI directory copied."
       
       # Copy all static pod manifests
       sudo cp -R /etc/kubernetes/manifests "$CONFIG_DIR/manifests"
       if [ $? -ne 0 ]; then
           echo "❌ ERROR: Failed to copy manifests directory."
           exit 1
       fi
       echo "✅ Manifests directory copied."
       
       # Copy the admin configuration file
       sudo cp /etc/kubernetes/admin.conf "$CONFIG_DIR/admin.conf"
       if [ $? -ne 0 ]; then
           echo "❌ ERROR: Failed to copy admin.conf."
           exit 1
       fi
       echo "✅ admin.conf copied."
       
       # Copy the current kubeadm configuration file
       sudo cp /etc/kubernetes/kubeadm-config.yaml "$CONFIG_DIR/kubeadm-config.yaml"
       if [ $? -ne 0 ]; then
           echo "❌ ERROR: Failed to copy kubeadm-config.yaml."
           exit 1
       fi
       echo "✅ kubeadm-config.yaml copied."
       
       # --- 5. Export All Kubernetes Resources ---
       
       RESOURCES_FILE="${CONFIG_DIR}/all-resources-pre-upgrade.yaml"
       echo "Exporting all Kubernetes resources to ${RESOURCES_FILE}..."
       
       # Export key cluster resources (all, configmaps, secrets, pv, pvc, etc.)
       kubectl get all,configmaps,secrets,pv,pvc,storageclass,networkpolicy,crds -A -o yaml > "$RESOURCES_FILE"
       if [ $? -ne 0 ]; then
           echo "❌ ERROR: Failed to export Kubernetes resources."
           exit 1
       fi
       
       if [ ! -s "$RESOURCES_FILE" ]; then
           echo "⚠️ WARNING: Kubernetes resources export file is empty. Check kubectl access."
       else
           echo "✅ All Kubernetes resources exported."
       fi
       
       echo "---"
       echo "🎉 **Backup Complete!** 🎉"
       echo "Backup location: ${BACKUP_DIR}"
       echo "etcd snapshot: ${LOCAL_PATH}"
       echo "Configuration files: ${CONFIG_DIR}"
    ```

  - Upgrade Script
    upgrade_k8s.sh

    ```sh
       #!/usr/bin/env bash
       
       # --- User Configuration ---
       # IMPORTANT: Set your desired target version here.
       # Use the full version, e.g., 1.33.9
       TARGET_K8S_VERSION="1.33.5"
       
       # Extract the major.minor part (e.g., 1.33 from 1.33.9)
       K8S_MAJOR_MINOR=$(echo "$TARGET_K8S_VERSION" | awk -F'.' '{print $1"."$2}')
       
       # --- 1. Initial Checks (jq and Root) ---
       
       # Check for root permissions
       if [[ $EUID -ne 0 ]]; then
          echo "❌ ERROR: This script must be run as root or with sudo."
          exit 1
       fi
       
       # Check if 'jq' is installed
       echo "Checking for 'jq' installation..."
       if ! dnf list installed jq &> /dev/null; then
           echo "❌ ERROR: 'jq' is not installed. Please install it with 'dnf install -y jq'."
           exit 2
       fi
       echo "✅ 'jq' is installed."
       
       # --- 2. Configure Kubernetes Repository ---
       
       K8S_REPO_FILE="/etc/yum.repos.d/kubernetes.repo"
       echo "Creating Kubernetes repo file: ${K8S_REPO_FILE}"
       
       # Note: Kubernetes repos use the Major.Minor version (v1.33)
       cat <<EOF | tee "$K8S_REPO_FILE"
       [kubernetes]
       name=Kubernetes
       baseurl=https://pkgs.k8s.io/core:/stable:/v${K8S_MAJOR_MINOR}/rpm/
       enabled=1
       gpgcheck=1
       gpgkey=https://pkgs.k8s.io/core:/stable:/v${K8S_MAJOR_MINOR}/rpm/repodata/repomd.xml.key
       exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
       EOF
       
       if [ $? -ne 0 ]; then
           echo "❌ ERROR: Failed to create/write ${K8S_REPO_FILE}."
           exit 1
       fi
       echo "✅ Kubernetes repo configured for v${K8S_MAJOR_MINOR}."
       
       # --- 3. Configure CRI-O Repository ---
       
       CRIO_REPO_FILE="/etc/yum.repos.d/cri-o.repo"
       echo "Creating CRI-O repo file: ${CRIO_REPO_FILE}"
       
       # Note: CRI-O repos also use the Major.Minor version (v1.33)
       cat <<EOF | tee "$CRIO_REPO_FILE"
       [cri-o]
       name=CRI-O
       baseurl=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/v${K8S_MAJOR_MINOR}/rpm/
       enabled=1
       gpgcheck=1
       gpgkey=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/v${K8S_MAJOR_MINOR}/rpm/repodata/repomd.xml.key
       exclude=cri-o
       EOF
       
       if [ $? -ne 0 ]; then
           echo "❌ ERROR: Failed to create/write ${CRIO_REPO_FILE}."
           exit 1
       fi
       echo "✅ CRI-O repo configured for v${K8S_MAJOR_MINOR}."
       
       # --- 4. Install/Update Components ---
       
       echo "Updating/Installing Kubernetes components to version ${TARGET_K8S_VERSION}..."
       
       # Install the specific version. The 'disableexcludes' is crucial.
       dnf install --assumeyes --disableexcludes=kubernetes,cri-o \
           kubeadm-"$TARGET_K8S_VERSION" \
           kubectl-"$TARGET_K8S_VERSION" \
           kubelet-"$TARGET_K8S_VERSION" \
           cri-o
       
       if [ $? -ne 0 ]; then
           echo "❌ ERROR: Failed to update one or more Kubernetes components."
           exit 1
       fi
       echo "✅ Components updated to version ${TARGET_K8S_VERSION}."
       
       # --- 5. Run Post-Installation Upgrade Commands (Based on Argument) ---
       
       # Check if an argument was provided (master or node)
       if [[ -z "$1" ]]; then
           echo "⚠️ WARNING: No argument provided (e.g., 'master' or 'node'). Skipping kubeadm upgrade steps."
       elif [[ "$1" == "master" ]]; then
           echo "--- MASTER UPGRADE STEPS ---"
           echo "Running checks for kubeadm upgrade plan v${TARGET_K8S_VERSION}"
           
           kubeadm upgrade plan "v${TARGET_K8S_VERSION}"
           if [ $? -ne 0 ]; then
               echo "❌ ERROR: 'kubeadm upgrade plan' failed. Review output before proceeding."
               exit 1
           fi
           
           echo "---"
           echo "👉 **NEXT STEP:** RUN THE UPGRADE COMMAND AS PRESCRIBED IN THE PLAN OUTPUT:"
           echo "   E.g., run 'kubeadm upgrade apply v${TARGET_K8S_VERSION}'"
           echo "👉 **FINALLY:** After 'kubeadm upgrade apply' succeeds, remember to restart the kubelet:"
           echo "   run 'systemctl daemon-reload'"
           echo "   run 'systemctl restart kubelet'"
       
       elif [[ "$1" == "node" ]]; then
           echo "--- NODE UPGRADE STEPS ---"
           
           kubeadm upgrade node
           if [ $? -ne 0 ]; then
               echo "❌ ERROR: 'kubeadm upgrade node' failed. Review output before proceeding."
               exit 1
           fi
           
           echo "Reloading daemon and restarting kubelet..."
           systemctl daemon-reload
           if [ $? -ne 0 ]; then
               echo "❌ ERROR: 'systemctl daemon-reload' failed."
               exit 1
           fi
           
           systemctl restart kubelet
           if [ $? -ne 0 ]; then
               echo "❌ ERROR: 'systemctl restart kubelet' failed."
               exit 1
           fi
           echo "✅ Node upgrade complete. Kubelet restarted."
       
       else
           echo "⚠️ WARNING: Unrecognized argument '$1'. Expected 'master' or 'node'."
       fi
       
       echo "---"
       echo "🎉 **Script Finished!**"
    ```
