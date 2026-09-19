---
title: cheat sheet/freeipa
tags: [cheat sheet]
---

- kerberos + hostgroups + dns via ipa
  - ```sh
    echo "<redacted>" | kinit admin
    ipa hostgroup-find --all --raw | grep "cn: "
    ipa hostgroup-show <group> --raw | awk -F "=|,|\\\\." '/member/ {print $2}'
    ipa dnsrecord-find <zone> --sizelimit=10000 --all --raw | awk '/cnamerecord/ && !/substitutionvariable/ { if (NR > 1) print prev; print; print "" } { prev = $0 }'
    ipa dnsrecord-find <zone> --sizelimit=10000 --all --raw | grep -A1 -E "idnsname: db[1-9][0-9]?-(ro|rw)"
    ```
