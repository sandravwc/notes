---
title: android/dotnet-arm64-proot
---

- .net 8 arm64 inside proot debian

  - ```sh
    curl -sSL https://dot.net/v1/dotnet-install.sh | bash /dev/stdin --channel 8.0
    export DOTNET_gcServer=0                 # GC init fails under proot's ptrace sandbox otherwise
    export DOTNET_GCHeapHardLimit=0x20000000
    apt install libicu-dev                   # dotnet --version fails without it
    ```

- porting an x64-only project

  - ```sh
    # check native deps (DllImport) are just standard apt-packaged libs, no x86 SIMD intrinsics -> RID override often just works
    dotnet publish -c Release -r linux-arm64 --self-contained false -p:RuntimeIdentifiers=linux-arm64
    # unversioned build (1.0.0.0) can break upstream version-compat checks -- inject real version like the project's own CI does:
    dotnet publish ... -p:Version=5.3.3 -p:InformationalVersion="5.3.3+<commit-sha>"   # quote it, msbuild splits InformationalVersion on unquoted commas
    ```
