---
title: cheat sheet/acme-autodns
---

- letsencrypt dns-01 via autodns (internetx), no port 80, no reverse proxy

  ```sh
  curl -s https://get.acme.sh | sh -s email=<mail>
  AUTODNS_USER=<api user> AUTODNS_PASSWORD=<pw> AUTODNS_CONTEXT=4 \
    ~/.acme.sh/acme.sh --issue --server letsencrypt --dns dns_autodns -d host.example.org
  ~/.acme.sh/acme.sh --install-cert -d host.example.org \
    --fullchain-file /path/fullchain.pem --key-file /path/key.pem --reloadcmd "<restart service>"
  # creds cached in ~/.acme.sh/account.conf (0600, SAVED_AUTODNS_*), own cron renews 4x/day, reloadcmd on renew
  # default ca is zerossl, --server letsencrypt to skip the EAB dance
  # context 4 = live system. other contexts -> "User does not exist or password incorrect"
  ```

- api user rights

  ```txt
  don't use the main login. make a user clone (Benutzerverwaltung -> clone), a plain sub-user sees no zones at all
  needs: zone read (0205) + zone update (0202) + zone BULK update (0202001)  <- the bulk one is what acme.sh uses, easy to miss
  EF00505 "User is not authorized for this function" = a right is missing
  S0205 summary=0 = user authenticates but owns/sees no zones
  ```

- debug

  ```sh
  acme.sh --issue ... --debug 2 2>&1 | grep autodns_response=
  # or raw:
  curl -s https://gateway.autodns.com -H 'Content-Type: text/xml' -d '<request><auth><user>U</user><password>P</password><context>4</context></auth><task><code>0205</code><view><limit>10</limit></view></task></request>'
  ```

- A record from a script, same bulk task

  ```xml
  <task><code>0202001</code>
    <default>
      <rr_rem><name>host</name><type>A</type><value>OLD</value></rr_rem>
      <rr_add><name>host</name><type>A</type><value>NEW</value><ttl>300</ttl></rr_add>
    </default>
    <zone><name>example.org</name><system_ns>a.ns14.net</system_ns></zone>
  </task>
  <!-- success = <type>success</type> in the response. dyndns = cron this with the current public ip -->
  ```

- public ip without a provider: router upnp first, echo service fallback

  ```sh
  # upnp: ssdp M-SEARCH for InternetGatewayDevice:1 -> LOCATION xml -> WANIPConnection controlURL -> SOAP GetExternalIPAddress
  # tp-link: Advanced -> NAT Forwarding -> UPnP, off by default
  curl -s https://ifconfig.co/ip || curl -s https://icanhazip.com
  # not from a box on a vpn, you get the vpn exit
  ```
