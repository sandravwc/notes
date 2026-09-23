---
title: cheat sheet/ssh-mesh
---

- ssh mesh, n machines any-to-any

    ```sh
    ssh-keygen                                          # same user key across machines you control is fine
    # append every machine's pubkey to every machine's authorized_keys
    ```

    ```sshconfig
    Host alias
        HostName 192.168.1.xxx
        Port 8022
        User someuser
        IdentityFile ~/.ssh/id_ed25519
    ```

    ```sh
    for h in a b c; do ssh -o BatchMode=yes -o ConnectTimeout=5 "$h" true && echo "$h ok" || echo "$h FAIL"; done
    ```
