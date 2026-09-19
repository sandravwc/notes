---
title: android/adb-input-automation
---

- drive an app via adb shell input

  - ```sh
    # screenshot coords need scaling to real device resolution (e.g. displayed 900x2000 for real 1080x2400 -> *1.2)
    adb shell input text 'single-quote the whole payload'   # ; > $ etc get eaten by adb's own remote shell otherwise
    # input text decodes literal %s as space -- breaks strings that already contain %s (e.g. printf '%s\n')
    # long strings: input text takes real time per char -- a following Enter can race ahead and split the command
    # ctrl+<key> via input keyevent --meta is unreliable -- tap the app's own on-screen modifier (e.g. termux CTRL), then send a plain KEYCODE_<x>
    # \mv -f, not mv -- aliased `mv -i` eats the next line as its y/n answer
    ```
