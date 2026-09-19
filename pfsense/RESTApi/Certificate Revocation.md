---
title: pfsense/RESTApi/Certificate Revocation
tags: [pfsense]
---

- get vpn certificates meta data to revoke
  ```sh
  curl --silent \
    --insecure \
    --header "X-API-Key:<redacted>" \
    https://<pfsense-host>:8005/api/v2/system/certificates \
    | jq '.data[] | select(.descr|test("user-a|user-b")) | {name: .descr, certref: .refid }'
  {
    "name": "vpn-user-a",
    "certref": "62568f43dd02a"
  }
  {
    "name": "vpn-user-b",
    "certref": "65ba5a8784811"
  }
  ```
- get appropriate crl
  ```sh
  curl --silent \
    --insecure \
    --header "X-API-Key:<redacted>" \
    https://<pfsense-host>:8005/api/v2/system/crls \
    | jq '.data[] | {"crl id": .id, "crl name": .descr}'
  {
    "crl id": 0,
    "crl name": "OpenVPN-Revocation"
  }
  ```
- add certificate to crl
  ```sh
  curl --silent \
    --insecure \
    --request "POST" \
    --header "X-API-Key:<redacted>" \
    --header "Content-Type: application/json" \
    --header "accept: application/json" \
    "https://<pfsense-host>:8005/api/v2/system/crl/revoked_certificate" \
    --data '
  {
    "parent_id": "0",
    "certref": "65ba5a8784811",
    "reason": -1
  }'
  
  {"code":200,"status":"ok","response_id":"SUCCESS","message":"","data":{"parent_id":"0","id":2,"certref":"65ba5a8784811","serial":null,"reason":-1,"revoke_time":1746180758}}
  ```
- double check if certs are in list
  ```sh
  curl --silent \
    --insecure \
    --header "X-API-Key:<redacted>" \
    https://<pfsense-host>:8005/api/v2/system/crl?id=0 \
    | jq '.data.cert[].certref'
  "634531e58dc17"
  "63fde1434226f"
  "65ba5a8784811"
  "62568f43dd02a"
  ```
