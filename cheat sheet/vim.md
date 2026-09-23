---
title: cheat sheet/vim
---

- paste current file name

  ```txt
  "%p
  ```

- add to end of line

  ```txt
  enter visual block mode.
  select lines
  $ - move cursor to last character
  A - enter insert mode after last character
  insert desired text
  <Esc> - exit insert mode and finish block append
  ```

- comment out a line range, and undo it

  ```txt
  :14,52s/^/#
  :14,52s/^#
  ```

- delete matching lines

  ```txt
  :g/^\;/d     lines starting with ;
  :g/^$/d      empty lines, sed equivalent: -r '/^\s*$/d'
  ```

- yank targets

  ```txt
  y$   to end of line
  y^   to start of line
  yw   to next word
  yiw  current word
  v / V / ctrl-v to select first
  ```
