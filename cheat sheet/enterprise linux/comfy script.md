---
title: cheat sheet/enterprise linux/comfy script
---

- scripts live in [comfy-env](https://github.com/sandravwc/comfy-env), edit heredocs before running
- atuin `sync_address` has to match atuin server url
- follow [[cheat sheet/enterprise linux/install atuin server]] to install atuin server to sync histories between servers
- enterprise linux (8|9) and higher generic: [el.sh](https://github.com/sandravwc/comfy-env/blob/master/el.sh)

  ```sh
  curl -fsSL https://github.com/sandravwc/comfy-env/archive/master.tar.gz | tar xz && bash comfy-env-master/el.sh
  ```

- enterprise linux 8 and higher zfs: [el-zfs.sh](https://github.com/sandravwc/comfy-env/blob/master/el-zfs.sh)

  ```sh
  curl -fsSL https://github.com/sandravwc/comfy-env/archive/master.tar.gz | tar xz && bash comfy-env-master/el-zfs.sh
  ```

- enterprise linux (8|9) kubernetes: [el-k8s.sh](https://github.com/sandravwc/comfy-env/blob/master/el-k8s.sh)

  ```sh
  curl -fsSL https://github.com/sandravwc/comfy-env/archive/master.tar.gz | tar xz && bash comfy-env-master/el-k8s.sh
  ```

- enterprise linux 7 generic: [el7.sh](https://github.com/sandravwc/comfy-env/blob/master/el7.sh)

  ```sh
  curl -fsSL https://github.com/sandravwc/comfy-env/archive/master.tar.gz | tar xz && bash comfy-env-master/el7.sh
  ```
