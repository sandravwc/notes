---
title: kubernetes/change storage class
tags: [kubernetes]
---

- assuming changing from longhorn to ceph
- assuming removal of longhorn
- assuming changing storage class name of a stateful set
  ```sh
  kubectl -nlonghorn-system edit settings.longhorn.io deleting-confirmation-flag
  helm uninstall -nlonghorn-system longhorn
  kubectl delete ns longhorn-system
  kubectl edit statefulset loki -nmonitoring
  kubectl get storageclass
  kubectl delete storageclass longhorn-static
  kubectl edit statefulset loki -nmonitoring
  
  # change storage class name
  vi /tmp/kubectl-edit-4100783938.yaml 
  
  kubectl delete statefulset loki -nmonitoring
  kubectl delete pvc storage-loki-0 -nmonitoring
  kubectl get pods -n monitoring
  kubectl apply -f /tmp/kubectl-edit-4100783938.yaml 
  kubectl get pods -n monitoring
    
  # check if works
  
  kubectl exec -n monitoring loki-0 -- df -hT /data
  Filesystem           Type            Size      Used Available Use% Mounted on
  /dev/rbd0            ext4            9.7G      3.0M      9.7G   0% /data
  ```
