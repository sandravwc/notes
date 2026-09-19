---
title: pacemaker/motd
---

- marek-motd

  ```sh
   _____________
  < F O O B A R >
   -------------
          \   ^__^
           \  (oo)\_______
              (__)\       )\/\
                  ||----w |
                  ||     ||
  ####pacemaker#####
  ### Prüfen: 
  allg. status:                           `pcs status`
  resources anschauen:                    `pcs resource status`
  resource config anschauen:              `pcs resource config $resource`
  
  ### Admin Stuff:
  resource bearbeiten:                    `pcs resource update $resource $option=$wert $option2=$wert2...`
  resource move:                          `pcs resource move $node`
  deactive/activate node:                 `pcs node standby $node` `pcs node unstandby $node`
  delete errors:                          `pcs resource clear $resource` `pcs resource cleanup`
  
  ### Score infinity if resource moved:
  show constraints:                       `pcs constraint show`
  clear for no failover after reboot:     `pcs resource clear $resource`
  
  ####drbd
  ### Prüfen: 
  allg. status:                           `drbdadm status`
  config anschauen:                       `cat /etc/drbd.d/drbd.res`
  ### Admin Stuff:
  config pushen:                          `pcs resource disable f_nfs_fs` && `drbdadm disconnect $resource` && `drbdadm adjust $resource`  ## ON BOTH NODES
  ### Fehler Lösen:                       `pcs resource disable f_nfs_fs`                                                                  ## DISABLE FS RESOURCE IN PCS FIRST THEN PROCEED
  status disconnected:                    `drbdadm secondary $resource` && `drdbadm (--discard-my-data) connect $resource`                 ## ON SLAVE
  funny splitbrain stuff:                 `drbdadm invalidate $resource` => GOTO md defect                                                 ## ON NODE BEHIND
  md defect:                              `drbdadm create-md $resource` && `drbdadm secondary $resource` && `drdbadm connect $resource`    ## ON DEFECT NODE
  corrupted file system on 1 node:        `mkfs.ext4 /dev/dbrd0` && `drdbadm (--discard-my-data) connect $resource`                        ## ON DEFECT NODE
  ```

- kilian-motd

  ```sh
  ######## Pacemaker Befehle
  
  - Status abrufen:
  pcs status
  
  - Fehler löschen 
  pcs resource cleanup
  
  - Resource auf anderen Host moven
  pcs resource move r_mysql_drbd-clone db02.example.internal --master
  
  - Host in Standby / Unstandby versetzen
  pcs node standby db02.example.internal 
  pcs node unstandby db01.example.internal
  
  
  ######## DRBD Fixen
  
  Sync Status prüfen:
  
  drbdadm status
  
  1. Auf dem Slave:
  
          drbdadm secondary mysql
          drbdadm connect (--discard-my-data) mysql
  
  * Das in Klammer geschriebene nur ausführen, wenn ein normaler connect nicht funktioniert; Forciert das ganze dann
  
  2. Auf dem Master:
  
          drbdadm connect mysql 
  
  ###############
  
  Wenn Ressource verschoben wird wird eine neuer Score INFINITY gesetzt:
  
  pcs constraint show
  
  Clearen mit (dass nach reboot kein auto failover getriggert wrid):
  
  pcs resource clear r_mysql_drbd-clone
  ```
