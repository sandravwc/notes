---
title: android/mealprep-server
---

- repo: <https://github.com/sandravwc/mealprep> -- receipt photo -> gemma 4 e4b on poco -> stock json -> daily recipe push. phases 0-3 live 2026-09-17, phase 4 = fridge photo + npu (see docs/TODO.md)
- poco services + cron

  - ```sh
    export SVDIR=$PREFIX/var/service   # non-login ssh needs it
    sv status mealprep crond           # mealprep: app.py :8090 http + :8443 https. llama service exists but disabled
    # logs: $PREFIX/var/log/sv/<name>/current
    crontab -l
    # */5 intake.py, */5 fridge.py, 16:50 janitor.py, */15 suggest.py --due (meals+times from data/config.json), acme.sh renew 4x/day
    ```

  - ```sh
    # runsvdir does not start on boot by itself
    cat ~/.termux/boot/10-services.sh
    # export SVDIR=...; . $PREFIX/etc/profile.d/start-services.sh
    ```

- deploy

  - ```sh
    cd ~/mealprep/repo && git pull && sv restart mealprep
    # intake/suggest/janitor are one-shot, pick up the pull on next run
    ```

- llama-server gemma 4 e4b vision

  - ```sh
    # no resident server anymore: intake.llm() starts this per job, stops it after (5 gb resident got termux killed twice)
    ./llama.cpp/build-cpu/bin/llama-server -m models/gemma-4-E4B-it-Q4_0.gguf --mmproj models/mmproj-gemma-4-E4B-it-Q8_0.gguf --jinja -t 8 -c 4096 --host 127.0.0.1 --port 8080
    # request must carry "chat_template_kwargs":{"enable_thinking":false} or gemma thinks until max_tokens
    # 8+ gen 1 cpu: 8 thr pp 24 / tg 4.9 tok/s, receipt ~90 s + 15 s load
    ```

- data + secrets

  - ```sh
    ~/mealprep/data/{inventory,receipts,suggestions,profile,config,aliases}.json
    ~/mealprep/receipts/*.jpg   # syncthing folder
    ~/mealprep/.env             # NTFY_TOPIC, BASE_URL, never in repo
    # aliases: "raw line lowercase": "canonical" | "" = drop
    # delete data/config.json -> defaults from server/config.py
    ```

- gpu/npu from termux: dead on 8+ gen 1 (sm8475), gpu works from adb shell

  - ```txt
    opencl: linker namespace blocks /vendor/lib64/libOpenCL.so for untrusted_app, opencl-vendor-driver shim exports nothing usable
    hexagon: llama.cpp htp libs v73+ only, 8+ gen 1 = v69
    litert+qnn lists sm8450 but apk only
    adb shell (shell uid) loads the vendor driver fine -> see block below. wireless debugging resets on reboot
    qwen3-vl-2b q8 on adreno 730: pp 62 / tg 13.9 vs cpu pp 88 / tg 11.5 (both under load). marginal
    ```

- gotchas

  - ```txt
    pkill -f <name> from an ssh one-liner kills the ssh bash itself (cmdline matches). use pkill -x
    backgrounding jobs inside ssh -c hangs the session, run ssh itself in background instead
    fcntl.flock(open(...)) without keeping the handle = no lock at all (gc closes it)
    ```

- adreno opencl from adb shell (termux can't, untrusted_app namespace)

  - ```sh
    # termux: stage binaries + every NEEDED lib + model into shared storage (both uids read /sdcard)
    cd ~/mealprep/llama.cpp/build-ocl/bin
    cp *.so* llama-bench llama-server $PREFIX/lib/libc++_shared.so $PREFIX/lib/libssl.so.3 $PREFIX/lib/libcrypto.so.3 model.gguf ~/storage/shared/llm/
    # readelf -d llama-bench | grep NEEDED  -> anything missing, copy from $PREFIX/lib
    
    # workstation (wireless debugging paired)
    adb shell 'cp /sdcard/llm/* /data/local/tmp/llm/ && chmod 755 /data/local/tmp/llm/*'
    adb shell 'rm /data/local/tmp/llm/libOpenCL.so'   # termux ocl-icd shadows the vendor driver -> "platform IDs not available"
    adb shell 'cd /data/local/tmp/llm && LD_LIBRARY_PATH=/data/local/tmp/llm:/vendor/lib64 ./llama-bench -m model.gguf -ngl 99'
    # ggml_opencl: device: QUALCOMM Adreno(TM) 730
    # wireless debugging is off after reboot, pairing survives
    ```

- gemma e4b vision quirks

  - ```txt
    small vlms parrot example product names from the prompt -> describe the rule, never list examples
    qwen3-vl-2b at temp 0 loops one word until max_tokens on cluttered photos -> unusable without repeat penalty
    --image-max-tokens 1024 on llama-server bounds qwen's native-res token blowup (1440x1920 photo = 300 s otherwise)
    8 gb phys + 4 gb swap: e4b resident 5 gb, second model beside it got termux killed
    ```

- https: acme.sh dns-01 via autodns, no reverse proxy

  - ```sh
    # api user = clone of main user in autodns. needs zone read + update + BULK update (0202001, what acme.sh uses)
    curl -s https://get.acme.sh | sh -s email=<mail>
    AUTODNS_USER=<clone> AUTODNS_PASSWORD=<pw> AUTODNS_CONTEXT=4 ~/.acme.sh/acme.sh --issue --server letsencrypt --dns dns_autodns -d poco.xn--bdk.dog
    ~/.acme.sh/acme.sh --install-cert -d poco.xn--bdk.dog --fullchain-file ~/mealprep/tls/fullchain.pem --key-file ~/mealprep/tls/key.pem --reloadcmd "SVDIR=$PREFIX/var/service sv restart mealprep"
    # creds cached in ~/.acme.sh/account.conf 0600, own cron renews. app.py wraps socket with stdlib ssl on 8443 when the files exist
    # A record with the LAN ip in public dns is fine. context 4 = live system, other contexts = auth error
    # debug: --debug 2, grep autodns_response=  (EF00505 = missing right, S0205 summary=0 = user sees no zones)
    ```

- termux killed by hyperos: diagnose, recover over adb

  - ```sh
    # ssh refused but phone pings = termux app process died
    adb shell dumpsys activity exit-info com.termux | grep -E 'reason=|description='   # LOW_MEMORY / OneKeyClean
    adb shell am start -n com.termux/.HomeActivity     # relaunch, profile.d starts runsvdir + services
    adb shell dumpsys deviceidle whitelist +com.termux
    adb shell cmd appops set com.termux RUN_IN_BACKGROUND allow
    # also: lock termux in recents, battery saver no restrictions. 8 gb phys + 4 gb swap, not 12
    # pkill -f <pattern> inside an ssh one-liner kills the ssh bash itself, use pkill -x
    ```

- tailscale on termux without root: built from upstream, works, dropped (client needs the app forever)

  - ```sh
    # official static binary dies: netmon.New: route ip+net: netlinkrib: permission denied (android blocks netlink for app uids)
    # fix lives in termux's go stdlib patch (net.Interfaces falls back on EPERM), so build with termux go:
    pkg install golang; git clone --depth 1 --branch v1.102.4 https://github.com/tailscale/tailscale ~/tailscale-src
    # remove !android from build lines in feature/acme feature/condregister ipn/localapi cmd/tailscale/cli, only inside //go:build lines
    sed -i -E '/^\/\/go:build/{s/!android && //; s/ && !android//; s/ \|\| android//}' <files>
    go list -tags "$TAGS" ./cmd/tailscaled   # check constraints in seconds before the 1-min compile
    go build -p 3 -trimpath -tags "$TAGS" -ldflags="-s -w" -o ~/.tailscale/ ./cmd/tailscaled ./cmd/tailscale   # TAGS = ts_omit_ssh,ts_omit_tap,... see ~/.tailscale/build-tags
    ~/.tailscale/tailscaled --tun=userspace-networking --statedir=~/.tailscale/state --socket=~/.tailscale/sock
    ```

- vlm findings for fridge photos (gemma 4 e4b)

  - ```txt
    recall on cluttered shelves ~50 %, names generic. german prompt, no example products (they get parroted), ask for {hinweis, items}
    2x2 tiles: +2 real labels, +3 invented per photo, 4x time -> rejected
    qwen3-vl-2b: parrots examples, loops one word at temp 0 -> rejected. never run two models at once on 8 gb
    sanity: regex drops container-only names, second text-only ask keeps food/drink only, no confidence numbers (uncalibrated)
    photo hint from model + rule (<3 kept or >half dropped) shown with warning in push and ui
    stickers get read as products ("Bon Jovi", "Placebo"), foil bundles are invisible
    ```
