---
title: hypervisors/proxmox/comfy script
---

- ```sh
  #!/usr/bin/env bash
  apt-get install neovim bash-completion bat git make gawk fzf -y
  curl -sSL https://alessandromrc.github.io/fastfetch-installer/installer.sh | bash
  mkdir -p /root/workdir
  git clone https://github.com/akinomyoga/ble.sh.git /root/workdir/ble.sh
  cd /root/workdir/ble.sh && make install INSDIR=/usr/local/lib/blesh
  
  cat <<- 'BLESHRC' > /root/.blerc
  
  ble-bind -f 'M-B' 'backward-cword'
  ble-bind -f 'M-F' 'forward-cword'
  bleopt complete_auto_complete=
  bleopt complete_auto_history=
  bleopt complete_ambiguous=
  bleopt prompt_eol_mark=''
  bleopt complete_menu_filter=
  bleopt history_share=1
  # bash-completion pre-escapes rsync/scp local paths (scp style), ble.sh would quote them a second time
  function my/scp-dequote-compreply {
    case ${COMP_WORDS[0]} in (rsync|scp) ;; (*) return 0 ;; esac
    ((${#COMPREPLY[@]})) || return 0
    # fzf wrapper calls the advised original -> runs twice per request
    [[ $_my_scp_dequoted == "$COMP_LINE:$COMP_POINT" ]] && return 0
    _my_scp_dequoted=$COMP_LINE:$COMP_POINT
    local i ret
    for i in "${!COMPREPLY[@]}"; do
      ble/syntax:bash/simple-word/eval "${COMPREPLY[i]% }" && COMPREPLY[i]=$ret
    done
  }
  function my/adjust-scp-completions {
    case $comp_func in
    (_comp_cmd_rsync|_comp_cmd_scp|_rsync|_scp|_fzf_path_completion)
      ble/function#advice after "$comp_func" my/scp-dequote-compreply ;;
    esac
  }
  blehook complete_load!='ble/function#advice after ble/complete/progcomp/adjust-third-party-completions my/adjust-scp-completions'
  BLESHRC
  
  cat <<- 'MOTDSH' > /etc/profile.d/motd.sh
  #!/usr/bin/env bash
  fastfetch
  MOTDSH
  
  cat <<- 'BASHRC' > /root/.bashrc
  # export paths and other variables first
  source "$HOME/.atuin/bin/env"
  export ATUIN_NOBIND="true"
  export PATH=$PATH:$HOME/bin
  export HISTTIMEFORMAT="%F %T "
  export HISTSIZE="100000"
  export LS_OPTIONS='--color=auto'
  
  if [[ $- == *i* ]] # execute if in interactive shell
  then
    PS1='\[\033[1;37m\][`date +%H:%M:%S`]\[\033[1;36m\][\[\033[1;31m\]\u\[\033[1;33m\]@\[\033[1;32m\]\h:\[\033[1;35m\]\w\[\033[1;36m\]]\[\033[1;31m\]\\$\[\033[0m\] '
    source -- /usr/local/lib/blesh/ble.sh --attach=none
    eval "$(atuin init bash)"
    bind -x '"\C-q": __atuin_history'
    source /usr/share/bash-completion/bash_completion
    source /usr/share/doc/fzf/examples/key-bindings.bash
    alias cat='batcat --style=plain --paging=never'
  fi
  
  umask 022
  alias rm='rm -i'
  alias cp='cp -i'
  alias mv='mv -i'
  alias ls='ls $LS_OPTIONS'
  alias ll='ls $LS_OPTIONS -l'
  alias l='ls $LS_OPTIONS -lA'
  alias ..='cd ..'
  alias ...='cd ../..'
  alias vi='nvim'
  
  if [[ $- == *i* ]] # execute if in interactive shell
  then
    [[ ! ${BLE_VERSION-} ]] || ble-attach
  fi
  BASHRC
  
  mkdir -p /root/.config/nvim
  
  cat <<- 'INITVIM' > /root/.config/nvim/init.vim
  set number
  set expandtab ts=2 sw=2 ai
  set listchars=eol:¬,tab:>·,trail:~,extends:>,precedes:<,space:␣
  set list
  set mouse=
  INITVIM
  
  curl --proto '=https' --tlsv1.2 -LsSf https://setup.atuin.sh | sh
  
  cat <<- 'ATUIN' > "/root/.config/atuin/config.toml"
  style = "full"
  enter_accept = true
  records = true
  auto_sync = true
  sync_frequency = 0
  sync_address = "http://10.0.1.6:8888"
  ATUIN
  
  cat <<- 'SCREENRC' > /root/.screenrc
  termcapinfo xterm* ti@:te@
  logfile "screenlog_%S.log"
  deflog on
  SCREENRC
  
  ```

-
-
