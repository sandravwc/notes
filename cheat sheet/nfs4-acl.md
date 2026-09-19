---
title: cheat sheet/nfs4-acl
tags: [cheat sheet]
---

- set nfsv4 acls
  - ```sh
    nfs4_setfacl -s "A::EVERYONE@:rxtncy,A:fdi:OWNER@:rwaDdxtTnNcCoy,A:fdi:EVERYONE@:rwaDdxtTnNcy,A::OWNER@:rwaDxtTnNcCoy,A:g:GROUP@:rxtncy" /export/assets
    ```
- compare acls between two nfs backends (migration)
  - ```sh
    for e in storage-a storage-b; do for d in a11y assets harbor jobs k8s logfiles mail sftp vhosts; do for a in {1..3}; do
      mkdir -p /"${e}"/"${d}"/acl-test/test0"${a}"; touch /"${e}"/"${d}"/acl-test/test0"${a}"/test-file
    done; done; done
    for d in a11y assets harbor jobs k8s logfiles mail sftp vhosts; do
      diff <(nfs4_getfacl /storage-a-acl/"${d}"/acl-test) <(nfs4_getfacl /storage-b-acl/"${d}"/acl-test)
      diff <(tree -pach /storage-a/"${d}"/acl-test) <(tree -pach /storage-b/"${d}"/acl-test)
    done
    # pipe through cat to keep color; watch for extra inherited ACEs (e.g. Administrators@BUILTIN) the target adds on its own
    ```
