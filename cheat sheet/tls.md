---
title: cheat sheet/tls
---

- expiry dates of a served cert, no file needed

  ```sh
  echo | openssl s_client -servername vhost.example.com -connect host.example.com:443 2>/dev/null | openssl x509 -noout -dates
  ```

- inspect a cert file

  ```sh
  openssl x509 -noout -text -in cert.crt
  openssl x509 -noout -dates -in cert.crt
  ```

- which ciphers a port actually offers (starttls aware)

  ```sh
  nmap --script ssl-enum-ciphers -p 25 host.example.com
  ```
