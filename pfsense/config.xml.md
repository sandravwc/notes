---
title: pfsense/config.xml
tags: [pfsense]
---

- xml in json für jq umwandeln
  ```sh
  yq -p=xml -o=json config-cust01-lb1.example.internal-20260219153714.xml > config-cust01-lb1.example.internal-20260219153714.json
  ```
- ipsec tunnel daten für leute aufbereiten
  ```sh
  jq -r '
    .pfsense.ipsec as $ipsec |
    ( $ipsec.phase2 | group_by(.ikeid) | map({key: .[0].ikeid, value: .}) | from_entries ) as $p2_map |
    $ipsec.phase1[] | . as $p1 |
    $p2_map[.ikeid][] |
    [
      $p1.descr,
      $p1."remote-gateway",
          if ($p1.peerid_data | length) == 0 then "None" else $p1.peerid_data end,
      if (.descr | length) == 0 then "None" else .descr end,
      .remoteid.address,
      .remoteid.netbits
    ] | @tsv
  ' config-cust02-lb1.example.internal-20250908144238.json     | column -s $'\t' -t
  
  jq '.pfsense.ipsec.phase1[] | select(.ikeid == "12")' config-cust02-lb1.example.internal-20250908144238.json > tunnel-12.new.json
  jq '.pfsense.ipsec.phase2[] | select(.ikeid == "12")' config-cust02-lb1.example.internal-20250908144238.json >> tunnel-12.new.json
  ```
