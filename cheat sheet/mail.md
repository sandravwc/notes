---
title: cheat sheet/mail
---

- send test mail from command line
  ```sh
  echo "sample message" | mail -r "$(hostname)" -s "sample mail subject" ops@example.com
  ```
- flush mailq
