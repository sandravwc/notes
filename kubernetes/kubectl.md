---
title: kubernetes/kubectl
---

- comfy to use: make yourself familiar with jq [[cheat sheet/jq]]
- select pods by their persistent volume claim
  this information can be then used to determine which persistent volumes are in use, for example to unmount orphaned mounts on a kubernetes node

  ```sh
  kubectl get pods -A -o=json | jq -c '.items[] | {name: .metadata.name, namespace: .metadata.namespace, claimName: (.spec.volumes[] | select(.persistentVolumeClaim? and (.persistentVolumeClaim.claimName | test("infinispan")) ).persistentVolumeClaim.claimName) }'
  {"name":"keycloak-infinispan-0","namespace":"kss-prod","claimName":"data-volume-keycloak-infinispan-0"}
  
  kubectl get pv -n kss-prod | grep infinispan
  pvc-294e1633-9962-4cac-856f-4744cd1474a9   1Gi        RWO            Delete           Bound         kss-prod/data-volume-keycloak-infinispan-1                 csi-rbd-hdd             537d
  pvc-60549316-4ee7-4f5b-8d50-c0f7acb3685d   1Gi        RWO            Delete           Bound         kss-prod/data-volume-keycloak-infinispan-0                 csi-rbd-hdd             537d
  pvc-8dca4d37-4225-4bce-ad30-c5396276fac9   1Gi        RWO            Delete           Bound         kss-prod/data-volume-keycloak-infinispan-2                 csi-rbd-hdd             537d
  
  kubectl get pod -n my-app-live -o=json  | jq -c '.items[] | {nre: .status.reason, name: .metadata.name, reason: .status.conditions[].message, container: .spec.containers[].name }'
  ```

- select pods with something in name (here: cassandra)

  ```sh
  kubectl get pods -A -ojson | jq -c '.items[] | select(.metadata.name|test("cassandra")) | {name: .metadata.name, namespace: .metadata.namespace, node: .spec.nodeName}'
  ```

- create a somewhat long lasting token (about 10years)

  ```sh
  kubectl create token -nargocd admin-user --duration="$((60**2*24*360*10))s"
  ```

- determine status of containers

  ```sh
  kubectl get pod -n my-app-live -o=json  | jq -c '.items[] | {nre: .status.reason, name: .metadata.name, reason: .status.conditions[].message, container: .spec.containers[].name }'
  
  possible output:
  {"nre":"Evicted","name":"my-app-traefik-live-77d644ddb7-7txkx","reason":"The node was low on resource: ephemeral-storage. Threshold quantity: 69899394982, available: 67930492Ki. Container traefik was using 52Ki, request is 0, has larger consumption of ephemeral-storage. Container fluentbit was using 29044Ki, request is 0, has larger consumption of ephemeral-storage. Container logrotate was using 7556Ki, request is 0, has larger consumption of ephemeral-storage. ","container":"traefik"}
  ```

- print a tls cert stored in a kubernetes secret

  ```sh
  kubectl get secret -nmy-namespace tls-wildcard.services.example.internal -ojson | jq -r '.data."tls.crt" | @base64d' | openssl x509 -inform pem -noout -text
  ```

- read a base64-encoded secret value directly

  ```sh
  kubectl get secret -n my-app-dev my-app-mysql-dev -o json | jq -r '.data["mysql-root-password"] | @base64d'
  ```

-
