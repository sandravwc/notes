---
title: pacemaker/mysql-cluster
---

- enterprise linux 8

  ```sh
  Schritte auf "srv01" ^ "srv02" ausführen.
  
  [13:37:00][root@srv01:~]$ echo '192.168.0.11 srv01 srv01.intern.domain.de' >> /etc/hosts
  [13:37:00][root@srv01:~]$ echo '192.168.0.12 srv02 srv02.intern.domain.de' >> /etc/hosts
  [13:37:00][root@srv01:~]$ yum install https://www.elrepo.org/elrepo-release-8.0-2.el8.elrepo.noarch.rpm
  [13:37:00][root@srv01:~]$ yum install kmod-drbd90 drbd90-utils nagios-plugins-drbd lvm2
  
  DRBD-Devices erstrecken sich über Logical Volumes.
  
  [13:37:00][root@srv01:~]$ pvcreate /dev/xvdb
  [13:37:00][root@srv01:~]$ vgcreate drbd /dev/xvdb
  [13:37:00][root@srv01:~]$ lvcreate -n data -l100%FREE drbd
  [13:37:00][root@srv01:~]$ touch /etc/drbd.d/drbd.res
  [13:37:00][root@srv01:~]$ ed /etc/drbd.d/drbd.res
  [13:37:00][root@srv01:~]$ cat /etc/drbd.d/drbd.res 
  resource mysql {
      volume 0 {
          device /dev/drbd0;
          disk /dev/drbd/data;
          meta-disk internal;
      }
  
      net {
          protocol C;
      }
  
      connection-mesh {
          hosts nfs01.intern.example.de nfs02.intern.example.de;
      }
  
      on nfs01.intern.example.de {
          address 192.168.12.41:7789;
          node-id 0;
      }
  
      on nfs02.intern.example.de {
          address 192.168.12.42:7789;
          node-id 1;
      }
  }
  [13:37:00][root@srv01:~]$ drbdadm create-md mysql
  [13:37:00][root@srv01:~]$ drbdadm up mysql
  [13:37:00][root@srv01:~]$ drbdadm primary --force mysql
  
  "drbdadm primary --force drbd01" nur auf dem Master ausführen
  
  [13:37:00][root@srv01:~]$ mkfs.ext4 /dev/drbd0
  [13:37:00][root@srv01:~]$
  [13:37:00][root@srv01:~]$
  [13:37:00][root@srv01:~]$ yum install pcs pacemaker --enablerepo=ha
  [13:37:00][root@srv01:~]$ passwd hacluster
  [13:37:00][root@srv01:~]$ systemctl enable pcsd
  [13:37:00][root@srv01:~]$ systemctl start pcsd
  [13:37:00][root@srv01:~]$ pcs cluster auth srv01.intern.domain.de srv02.intern.domain.de
  [13:37:00][root@srv01:~]$ pcs cluster setup mysqlha --start srv01.intern.domain.de srv02.intern.domain.de
  [13:37:00][root@srv01:~]$ pcs cluster start --all
  [13:37:00][root@srv01:~]$ pcs cluster enable --all
  [13:37:00][root@srv01:~]$ pcs stonith list
  [13:37:00][root@srv01:~]$ pcs property set stonith-enabled=false
  [13:37:00][root@srv01:~]$ pcs property show stonith-enabled
  [13:37:00][root@srv01:~]$ crm_verify -LV
  [13:37:00][root@srv01:~]$ pcs property list --all | grep quorum
  [13:37:00][root@srv01:~]$ pcs property set no-quorum-policy=ignore
  [13:37:00][root@srv01:~]$ pcs constraint list
  [13:37:00][root@srv01:~]$ pcs status
  
  Installation MySQL
  
  [13:37:00][root@srv01:~]$ pcs cluster cib mysql_cluster
  [13:37:00][root@srv01:~]$ pcs -f mysql_cluster property set no-quorum-policy=ignore
  [13:37:00][root@srv01:~]$ pcs -f mysql_cluster property set stonith-enabled=false
  [13:37:00][root@srv01:~]$ pcs -f mysql_cluster resource defaults resource-stickiness=200
  [13:37:00][root@srv01:~]$ pcs -f mysql_cluster resource create r_mysql_drbd ocf:linbit:drbd drbd_resource=mysql op monitor interval=10s
  [13:37:00][root@srv01:~]$ pcs -f mysql_cluster resource promotable r_mysql_drbd promoted-max=1 promoted-node-max=1 clone-max=2 clone-node-max=1 notify=true
  [13:37:00][root@srv01:~]$ pcs -f mysql_cluster resource status
  [13:37:00][root@srv01:~]$ pcs -f mysql_cluster resource create r_mysql_fs ocf:heartbeat:Filesystem device=/dev/drbd0 directory=/var/lib/mysql fstype=ext4
  [13:37:00][root@srv01:~]$ pcs -f mysql_cluster resource create r_mysqlserver ocf:heartbeat:mysql \
    binary="/usr/libexec/mysqld" \
    config="/etc/my.cnf" \
    datadir="/var/lib/mysql" \
    pid="/var/lib/mysql/mysql.pid" \
    socket="/var/lib/mysql/mysql.sock" \
    log="/var/log/mysql/mysqld.log" \
    additional_parameters="--bind-address=0.0.0.0" \
    op start timeout=60s \
    op stop timeout=60s \
    op monitor interval=20s timeout=30s
  [13:37:00][root@srv01:~]$ pcs -f mysql_cluster resource create r_mysqlip ocf:heartbeat:IPaddr2 ip=192.168.0.10 cidr_netmask=24 nic=eth0
  [13:37:00][root@srv01:~]$ pcs -f mysql_cluster resource group add g_mysql r_mysql_fs r_mysqlserver r_mysqlip
  [13:37:00][root@srv01:~]$ pcs -f mysql_cluster constraint colocation add g_mysql with r_mysql_drbd-clone INFINITY with-rsc-role=Master
  [13:37:00][root@srv01:~]$ pcs -f mysql_cluster constraint order promote r_mysql_drbd-clone then start g_mysql
  [13:37:00][root@srv01:~]$ pcs -f mysql_cluster resource status
  [13:37:00][root@srv01:~]$ pcs cluster cib-push mysql_cluster
  [13:37:00][root@srv01:~]$ pcs resource cleanup
  [13:37:00][root@srv01:~]$ pcs status
  
  Beim initialen mysql Setup wird es vermutlich passieren, dass mysql nicht gestartet werden kann mit dem Eintrag im Log, dass datadir nicht gefunden werden kann. 
  Hierfür tue:
  
  [13:37:00][root@srv01:~]$ pcs resource disable r_mysqlserver
  [13:37:00][root@srv01:~]$ chown -R mysql:mysql /var/lib/mysql && chmod -R 770 /var/lib/mysql
  [13:37:00][root@srv01:~]$ systemctl start mysqld
  [13:37:00][root@srv01:~]$ systemctl stop mysqld
  [13:37:00][root@srv01:~]$ pcs resource enable r_mysqlserver
  
  Dann noch NRPE-Plugins set-up-pen
  
  [13:37:00][root@srv01:~]$ yum install perl-Monitoring-Plugin.noarch perl-Params-Validate.x86_64 perl-Math-Calc-Units.noarch perl-Math-BigInt-FastCalc.x86_64 perl-Class-Accessor.noarch perl-Config-Tiny.noarch
  [13:37:00][root@srv01:~]$ sed  's/command\[check_drbd\]=/#command\[check_drbd\]=/g' /etc/nagios/nrpe_local.cfg
  [13:37:00][root@srv01:~]$ echo 'command[check_drbd]=/usr/lib/nagios/plugins/check_drbd9 -d all' >> /etc/nagios/nrpe_local.cfg
  [13:37:00][root@srv01:~]$ echo 'command[check_crm]=/usr/lib/nagios/plugins/check_crm' >> /etc/nagios/nrpe_local.cfg
  
  Scripts kann man sich abholen in: https://git.example.internal/ops/nagios-pacemaker
  
  [13:37:00][root@srv01:~]$ vi /usr/lib/nagios/plugins/check_drbd9
  [13:37:00][root@srv01:~]$ chmod +x /usr/lib/nagios/plugins/check_drbd9
  [13:37:00][root@srv01:~]$ nano /usr/lib/nagios/plugins/check_crm
  [13:37:00][root@srv01:~]$ chmod +x /usr/lib/nagios/plugins/check_crm
  [13:37:00][root@srv01:~]$ systemctl restart xinetd
  
  Dann noch Monitorings in SIMPL eintragen: check_drbd9, check_crm, check_corosync
  
  Und optional für Rebootpersistenz:
  
  [13:37:00][root@srv01:~]$ systemctl enable corosync.service
  [13:37:00][root@srv01:~]$ systemctl enable pacemaker.service
  ```
