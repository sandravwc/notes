---
title: cheat sheet/mail
tags: [cheat sheet]
---

- send test mail from command line
  ```sh
  echo "sample message" | mail -r "$(hostname)" -s "sample mail subject" ops@example.com
  ```
- flush mailq
