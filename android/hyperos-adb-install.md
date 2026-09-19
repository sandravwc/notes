---
title: android/hyperos-adb-install
---

- sideload apk on hyperos

  - ```sh
    # enable Settings > Additional settings > Developer options > "USB debugging (Security settings)" too (separate from plain USB debugging)
    # phone must be unlocked+onscreen -- AdbInstallActivity confirmation can't render over lockscreen
    # adb install -> INSTALL_FAILED_USER_RESTRICTED / streamed-parse bug regardless -- push + tap instead:
    adb push app.apk /sdcard/Download/app.apk
    # tap it in Files app (normal installer path, bypasses AdbInstallActivity)
    ```

- disable install-time scan

  - ```sh
    adb shell settings put global package_verifier_enable 0        # play protect, reversible (1 to revert)
    adb shell settings put global verifier_verify_adb_installs 0
    # xiaomi's own scanner (AdbInstallActivity etc, inside com.miui.securitycenter) is anti-tamper protected, not touchable without root
    ```

- drive tap-to-install via adb

  - ```sh
    adb shell content query --uri content://media/external/downloads --projection _id --where "_display_name='app.apk'"
    adb shell am start -a android.intent.action.VIEW -d content://media/external/downloads/<id> -t application/vnd.android.package-archive
    ```
