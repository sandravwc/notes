---
title: cheat sheet/unsorted
---

- ### here goes everything b4 being sorted
- yolo monitoring bulk change
  ```sh
  awk '!/^#/ && /k8s[2-3]?-(node|master)/ {print $2}' /etc/motd | xargs -I % ssh -n % "sed -i '/zombie/ s/-w 5 -c 10/-w 40 -c 50/' /etc/nagios/nrpe_local.cfg"
  ```
  - fix missing k8s kubekonfig
    ```sh
    [21:52:20][root@k8s-master-dev01:~]$ cp /etc/kubernetes/admin.conf /root/.kube/config
    [21:53:21][root@k8s-master-dev01:~]$ kubectl get pods -A
    ```
  - example custom repo layout
    ```sh
    [custom]
    name=Custom Development repository
    type=rpm-md
    metadata_expire=0
    http_caching=packages
    baseurl=https://repo.example.internal/nexus/content/repositories/company.releases/
    gpgcheck=0
    enabled=0
    - [AmazonCorretto]
    name=Amazon Corretto
    baseurl=https://yum.corretto.aws/$basearch
    enabled=1
    gpgkey=https://yum.corretto.aws/corretto.key
    gpgcheck=1
    ```
  - unsorted unsorted :D
    ```sh
    vi:
    :14,52s/^/#
    :14,52s/^#
    :g/^\;/d
    :g/^$/d empty lines // -r '/^\s*$/d'
    y$, y^, yw (end at next word), yiw (current word) // v,V,ctrlv
    - c8 vault:
    sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-Linux-*
    sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.epel.cloud|g' /etc/yum.repos.d/CentOS-Linux-*
    - colors:
    for ID1 in {1..50}; do for ID in {1..256}; do printf "[${ID1};5;${ID}m %s\n" "${ID1} ${ID}"; done; done
    - perconer:
    https://repo.percona.com/yum/percona-release-latest.noarch.rpm
    - zeht-ehf-ehs: 
    a='zfs get compression,recordsize,atime,sync,primarycache,secondarycache'; for pool in $(zfs list | awk '/zmysql/ {print $1}'); do eval $a $pool; echo; done
    - ze-sl: 
    echo | openssl s_client -servername host2.example.internal -connect host.example.internal:443 2>/dev/null | openssl x509 -noout -dates
    nmap --script ssl-enum-ciphers -p 25 203.0.113.10
    - rpm aufrupfen:
    rpm2cpio foo.rpm | cpio -idmv
    - elk?
    curl -XPOST 10.0.2.131:9200/_cluster/reroute?retry_failed=true
    curl -XPOST 10.0.2.131:9200/_cluster/allocation/explain
    - platzverbrauch von einzelnen sachen:
    [10:52:45][root@db01-production-master:/var/lib/mysql]$ echo $(($(l | awk '/mysql-bin/ {print $5}' | paste -s -d "+"  | bc)/2**30))
    50
    - [17:36:09][root@db02:/var/lib/mysql]$ echo $(($(l *-bin.* | awk '{print $5}' | paste -s -d "+" | bc )/2**30))GiB
    114GiB
    - 5598  2023-05-15 11:28:32 svn update
    5610  2023-05-15 11:30:22 svn info
    5618  2023-05-15 11:36:51 svn commit -m "a"
    5619  2023-05-15 11:36:57 svn co https://svn.example.internal/repo --username=myuser
    ```
