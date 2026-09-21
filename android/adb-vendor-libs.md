---
title: android/adb-vendor-libs
---

- run a binary with vendor libs (opencl, adreno) that termux can't load

  ```sh
  # termux = untrusted_app, linker namespace hides /vendor/lib64. adb shell = uid shell, allowed
  # 1. termux: stage binary + every NEEDED lib + data into shared storage
  cp bin/* $PREFIX/lib/libc++_shared.so $PREFIX/lib/libssl.so.3 $PREFIX/lib/libcrypto.so.3 ~/storage/shared/x/
  readelf -d bin/foo | grep NEEDED          # copy anything else it names from $PREFIX/lib
  # 2. workstation, wireless debugging paired
  adb shell 'cp /sdcard/x/* /data/local/tmp/x/ && chmod 755 /data/local/tmp/x/*'
  adb shell 'rm /data/local/tmp/x/libOpenCL.so'   # termux's ocl-icd loader shadows the vendor driver -> "platform IDs not available"
  adb shell 'cd /data/local/tmp/x && LD_LIBRARY_PATH=/data/local/tmp/x:/vendor/lib64 ./foo'
  # wireless debugging turns off on reboot, pairing survives. /data/local/tmp survives too
  ```

- what's there

  ```sh
  adb shell 'ls /vendor/lib64/libOpenCL.so /vendor/lib64/libcdsprpc.so; getprop ro.soc.model'
  # SM8475 = 8+ gen 1: adreno 730 opencl works this way
  # hexagon v69: llama.cpp ships HTP v73+ only -> no. executorch + qnn -> yes, verified, see [[android/hexagon-npu-llm]]
  ```
