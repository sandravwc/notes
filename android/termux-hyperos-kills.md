---
title: android/termux-hyperos-kills
---

- ssh refused, phone pings = termux app process died

  ```sh
  adb shell dumpsys activity exit-info com.termux | grep -E 'reason=|description='
  # reason=3 (LOW_MEMORY)            memory pressure, whole app judged by its rss (proot + a 5 gb model = gone)
  # reason=13 description=OneKeyClean hyperos "clean" button / auto clean
  adb shell am start -n com.termux/.HomeActivity   # relaunch, profile.d start-services brings runsvdir + services back
  # termux:boot only fires on reboot, not on app death
  ```

- keep it alive

  ```sh
  adb shell dumpsys deviceidle whitelist +com.termux
  adb shell cmd appops set com.termux RUN_IN_BACKGROUND allow
  adb shell cmd appops set com.termux RUN_ANY_IN_BACKGROUND allow
  # ui: recents -> lock termux card; settings -> apps -> termux -> battery saver -> no restrictions
  # don't sit on ram: start heavy processes per job and stop them after. 12 gb "ram" = 8 phys + 4 zram swap
  ```

- ssh one-liner gotchas

  ```sh
  pkill -x llama-server     # pkill -f <pattern> matches the ssh bash -c cmdline itself and kills your session
  # backgrounding a job inside `ssh host 'cmd &'` hangs the session until the job exits, background the ssh instead
  # non-login shells have no $SVDIR: export SVDIR=$PREFIX/var/service before sv
  ```
