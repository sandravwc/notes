---
title: cheat sheet/enterprise linux/install atuin server
---

- follow this after installing atuin
  - <https://docs.atuin.sh/self-hosting/server-setup>
- copy pasta

  - ```sh
    #!/usr/bin/env bash
    dnf install sqlite -y
    mkdir -p /etc/atuin
    
    cat <<- 'ATUIN' > /etc/atuin/server.toml
    host = "0.0.0.0"
    port = 8888
    open_registration = true
    db_uri="sqlite:///config/atuin.db"
    ATUIN
    
    mkdir -p /config
    sqlite3 /config/atuin.db
    
    
    cat <<- 'ATUIN_SERVICE' > /etc/systemd/system/atuin-server.service
    [Unit]
    Description=Start the Atuin server syncing service
    After=network-online.target
    Wants=network-online.target systemd-networkd-wait-online.service
    
    [Service]
    ExecStart=/root/.atuin/bin/atuin server start
    Restart=on-failure
    User=root
    Group=root
    
    Environment=ATUIN_CONFIG_DIR=/etc/atuin
    ReadWritePaths=/etc/atuin
    
    # Hardening options
    CapabilityBoundingSet=
    AmbientCapabilities=
    NoNewPrivileges=true
    ProtectHome=true
    ProtectSystem=strict
    ProtectKernelTunables=true
    ProtectKernelModules=true
    ProtectControlGroups=true
    PrivateTmp=true
    PrivateDevices=true
    LockPersonality=true
    
    [Install]
    WantedBy=multi-user.target
    ATUIN_SERVICE
    ```
