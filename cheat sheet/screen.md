---
title: cheat sheet/screen
tags: [cheat sheet]
---

- scroll in screen
  ```txt
  - Hit your screen prefix combination (C-a / control+A by default), then hit Escape or [.
  - Move up and down with the arrow keys (↑ and ↓).
  - When you're done, hit any key except arrow keys, numbers, and certain letters to get back to the end
  of the scroll buffer. Most people use q or Escape
  ```
  alternatively:
  ```sh
  echo "termcapinfo xterm* ti@:te@" >> /root/.screenrc
  ```
