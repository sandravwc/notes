---
title: android/adb-debloat
---

- adb debloat, non-rooted

  - ```sh
    adb shell pm uninstall --user 0 <pkg>          # reversible, per-user
    adb shell cmd package install-existing <pkg>   # restore
    adb shell pm disable-user --user 0 <pkg>       # fallback if uninstall refused
    adb shell pm list packages -f > packages.txt   # cross-check before removing anything
    ```

- never touch
  - SystemUI, Settings, com.android.shell, resource overlays, wifi/bt stack, play services/store, keychain, vpn dialogs
- avf/pkvm packages (if planning linux-in-avf later)
  - virtualmachine.res, compos.payload, microdroid.empty_payload, dynsystem -- check `/dev/kvm` exists first, some chipsets (snapdragon poco f5 pro) don't have it at all
- huawei: swipe-to-kill stops actually killing after debloat

  - ```sh
    # com.huawei.systemmanager owns swipe-kill in Recents (PROCESS_OPTIMIZE intent) -- removed it -> card vanishes, process stays alive
    adb shell cmd package install-existing com.huawei.systemmanager
    adb shell pidof <process>   # verify actually dead after a real swipe
    ```

- xiaomi: getapps ad spam

  - ```sh
    adb shell pm uninstall --user 0 com.xiaomi.mipicks   # "GetApps", pushes install-a-game notifications
    # com.xiaomi.xmsf/xmsfkeeper is shared push transport -- only kill if spam continues after mipicks is gone
    ```
