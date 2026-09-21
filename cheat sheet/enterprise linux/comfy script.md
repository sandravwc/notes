---
title: cheat sheet/enterprise linux/comfy script
---

- atuin `sync_address` has to match atuin server url
- follow [[cheat sheet/enterprise linux/install atuin server]] to install atuin server to sync histories between servers
- enterprise linux (8|9) and higher generic

  ```sh
  #!/usr/bin/env bash
  hostname=$(hostname)
  source /etc/os-release
  MAJOR_VERSION="${VERSION_ID%%.*}"
  if [[ "$MAJOR_VERSION" == "8" ]]; then
      powertools="powertools"
  elif [[ "$MAJOR_VERSION" == "9" ]]; then
      powertools="crb"
  else
      echo "Error: Unsupported AlmaLinux major version ($MAJOR_VERSION)." >&2
      exit 1
  fi
  dnf install --assumeyes --enablerepo="${powertools}" \
    btop \
    fzf \
    bat \
    neovim \
    bash-completion \
    fastfetch \
    glibc-langpack-en \
    git \
    dnf-automatic \
    nagios-plugins-check-updates \
    perl-Params-Validate \
    perl-Math-Calc-Units \
    perl-Class-Accessor \
    perl-Config-Tiny
  
  mkdir -p /root/.config/nvim
  mkdir -p /root/workdir
  mkdir -p /root/bin
  git clone https://github.com/akinomyoga/ble.sh.git /root/workdir/ble.sh
  cd /root/workdir/ble.sh && make install INSDIR=/usr/local/lib/blesh
  curl --proto '=https' --tlsv1.2 -LsSf https://setup.atuin.sh | sh
  
  cat <<- 'BASH_PROFILE' > /root/.bash_profile
  # exports paths and other variables first
  source "$HOME/.atuin/bin/env"
  export ATUIN_NOBIND="true"
  export PATH=$PATH:$HOME/bin
  export SYSTEMD_EDITOR=nvim
  export LANG=en_US.UTF-8
  export LC_ALL=en_US.UTF-8
  export HISTTIMEFORMAT="%F %T "
  export HISTSIZE="100000"
  export LS_OPTIONS='--color=auto'
  # and then aliases and such
  if [[ -f ~/.bashrc ]]; then
    source ~/.bashrc
  fi
  BASH_PROFILE
  
  cat <<- 'BASHRC' > /root/.bashrc
  if [[ $- == *i* ]] # execute if in interactive shell
  then
    PS1='\[\033[1;37m\][`date +%H:%M:%S`]\[\033[1;36m\][\[\033[1;31m\]\u\[\033[1;33m\]@\[\033[1;32m\]\h:\[\033[1;35m\]\w\[\033[1;36m\]]\[\033[1;31m\]\\$\[\033[0m\] '
    shopt -s extglob
    source -- /usr/local/lib/blesh/ble.sh --attach=none
    eval "$(atuin init bash)"
    bind -x '"\C-q": __atuin_history'
    source /usr/share/fzf/shell/key-bindings.bash
    alias cat='bat --style=plain --paging=never'
    if [[ $PS1 && -f /usr/share/bash-completion/bash_completion ]]
    then
      source /usr/share/bash-completion/bash_completion
    fi
  fi
  
  umask 022
  alias ls='ls $LS_OPTIONS'
  alias ll='ls $LS_OPTIONS -l'
  alias l='ls $LS_OPTIONS -lA'
  alias ..='cd ..'
  alias ...='cd ../..'
  alias vi='nvim'
  alias rm='rm -i'
  alias cp='cp -i'
  alias mv='mv -i'
  
  if [[ $- == *i* ]] # execute if in interactive shell
  then
   [[ ! ${BLE_VERSION-} ]] || ble-attach
  fi
  BASHRC
  
  cat <<- 'INITVIM' > /root/.config/nvim/init.vim
  set number
  set expandtab ts=2 sw=2 ai
  set listchars=eol:¬,tab:>·,trail:~,extends:>,precedes:<,space:␣
  set list
  set mouse=
  INITVIM
  
  cat <<- 'MOTDSH' > /etc/profile.d/motd.sh
  #!/usr/bin/env bash
  fastfetch \
    --logo none \
    --structure kernel:os:packages:uptime:loadavg:memory:cpu:disk:shell:localip:publicip
  MOTDSH
  
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

  # complete_requote_threshold compares the quoted length with the escaped one; a single special char loses by one -> discount the two quotes
  function my/patch-requote-threshold {
    local def; def=$(declare -f ble/complete/action/requote-final-insert)
    builtin eval -- "${def/'((${#ret}+threshold<=${#ins}))'/'((${#ret}-2+threshold<=${#ins}))'}"
  }

  # ambiguous completion inserts the common prefix backslash-escaped, requote as an open '...
  function my/requote-common-insert {
    [[ " ${ADVICE_FUNCNAME[*]} " == *' ble/complete/insert-common '* ]] && ((cand_count>1)) || return 0
    local word=${ADVICE_WORDS[3]}
    [[ $word != "$COMPS" && $word == *\\* && $COMPS != *[\\\$\`~=:{]* && $comps_flags != *[SEDI]* ]] || return 0
    local ret simple_flags simple_ibrace count
    ble/syntax:bash/simple-word/reconstruct-incomplete-word "$word" &&
      ble/complete/source/eval-simple-word "$ret" single:count && ((count==1)) || return 0
    local q=\' Q="'\\''"
    ADVICE_WORDS[3]=$q${ret//$q/$Q}
  }
  blehook complete_load!='my/patch-requote-threshold; ble/function#advice before ble/complete/insert my/requote-common-insert'

  # menu-complete (tab-tab cycling) inserts the raw escaped candidate, requote like the final insert
  function my/requote-menu-selection {
    local nsel=${ADVICE_WORDS[1]}
    ((nsel>=0)) && [[ :$bleopt_complete_menu_complete_opts: == *:insert-selection:* ]] || return 0
    local COMP1=${_ble_complete_menu0_comp[0]} COMP2=${_ble_complete_menu0_comp[1]}
    local COMPS=${_ble_complete_menu0_comp[2]} COMPV=${_ble_complete_menu0_comp[3]}
    local comp_type=${_ble_complete_menu0_comp[4]} comps_flags=${_ble_complete_menu0_comp[5]} comps_fixed=${_ble_complete_menu0_comp[6]}
    local "${_ble_complete_cand_varnames[@]/%/=}"
    ble/complete/cand/unpack "${_ble_complete_menu_items[nsel]}"
    local insert=$INSERT insert_flags= suffix=
    ble/complete/action/requote-final-insert
    [[ $insert != "$INSERT" ]] || return 0
    ble-edit/content/replace-limited "$_ble_complete_menu0_beg" "$_ble_edit_ind" "$insert"
    ((_ble_edit_ind=_ble_complete_menu0_beg+${#insert}))
  }
  blehook complete_load!='ble/function#advice after ble/complete/menu-complete.class/onselect my/requote-menu-selection'
  ble-import -d integration/fzf-completion
  ble-import -d integration/fzf-key-bindings
  BLESHRC
  
  cat <<- 'ATUIN' > "/root/.config/atuin/config.toml"
  style = "full"
  enter_accept = true
  records = true
  auto_sync = true
  sync_frequency = 0
  sync_address = "http://10.0.1.6:8888"
  ATUIN
  
  cat <<- DNFAUTOMATIC > /etc/dnf/automatic.conf
  [commands]
  upgrade_type = default
  random_sleep = 0
  network_online_timeout = 60
  download_updates = yes
  apply_updates = yes
  reboot = never
  reboot_command = "shutdown -r +5 'Rebooting after applying package updates'"
  [emitters]
  emit_via = stdio,email
  [email]
  email_from = root@${hostname}
  email_to = ops@example.com
  email_host = localhost
  [command]
  [command_email]
  [base]
  debuglevel = 2
  DNFAUTOMATIC
  
  mkdir -p /etc/systemd/system/dnf-automatic.timer.d
  
  cat <<- 'DNFAUTOMATICTIMER' > /etc/systemd/system/dnf-automatic.timer.d/override.conf
  [Timer]
  RandomizedDelaySec=28800
  OnCalendar=*-*-* 07:00:00
  DNFAUTOMATICTIMER
  
  cat <<- 'SCREENRC' > /root/.screenrc
  termcapinfo xterm* ti@:te@
  logfile "screenlog_%S.log"
  deflog on
  SCREENRC
  
  systemctl enable --now dnf.automatic.timer
  echo "command[check_security_updates]=/usr/lib64/nagios/plugins/check_updates --quiet --security-only" >> /etc/nagios/nrpe_local.cfg
  rm -f /root/anaconda-ks.cfg /root/changelog.txt /root/original-ks.cfg /root/postinstall.sh 
  rm -rf /root/workdir/ble.sh
  ```

- enterprise linux 8 and higher zfs

  ```sh
  #!/usr/bin/env bash
  dnf install --assumeyes --enablerepo=powertools \
    btop  \
    fzf \
    bat \
    neovim \
    bash-completion \
    fastfetch \
    glibc-langpack-en \
    git \
    dnf-automatic \
    nagios-plugins-check-updates \
    perl-Params-Validate \
    perl-Math-Calc-Units \
    perl-Class-Accessor \
    perl-Config-Tiny
  
  mkdir -p /root/.config/nvim
  mkdir -p /root/workdir
  mkdir -p /root/bin
  git clone https://github.com/akinomyoga/ble.sh.git /root/workdir/ble.sh
  cd /root/workdir/ble.sh && make install INSDIR=/usr/local/lib/blesh
  curl --proto '=https' --tlsv1.2 -LsSf https://setup.atuin.sh | sh
  
  cat <<- 'BASH_PROFILE' > /root/.bash_profile
  # exports paths and other variables first
  source "$HOME/.atuin/bin/env"
  export ATUIN_NOBIND="true"
  export PATH=$PATH:$HOME/bin
  export SYSTEMD_EDITOR=nvim
  export LANG=en_US.UTF-8
  export LC_ALL=en_US.UTF-8
  export HISTTIMEFORMAT="%F %T "
  export HISTSIZE="100000"
  export LS_OPTIONS='--color=auto'
  # and then aliases and such
  if [[ -f ~/.bashrc ]]; then
    source ~/.bashrc
  fi
  BASH_PROFILE
  
  cat <<- 'BASHRC' > /root/.bashrc
  if [[ $- == *i* ]] # execute if in interactive shell
  then
    PS1='\[\033[1;37m\][`date +%H:%M:%S`]\[\033[1;36m\][\[\033[1;31m\]\u\[\033[1;33m\]@\[\033[1;32m\]\h:\[\033[1;35m\]\w\[\033[1;36m\]]\[\033[1;31m\]\\$\[\033[0m\] '
    shopt -s extglob
    source -- /usr/local/lib/blesh/ble.sh --attach=none
    eval "$(atuin init bash)"
    bind -x '"\C-q": __atuin_history'
    source /usr/share/fzf/shell/key-bindings.bash
    alias cat='bat --style=plain --paging=never'
    if [[ $PS1 && -f /usr/share/bash-completion/bash_completion ]]
    then
      source /usr/share/bash-completion/bash_completion
    fi
  fi
  
  umask 022
  alias ls='ls $LS_OPTIONS'
  alias ll='ls $LS_OPTIONS -l'
  alias l='ls $LS_OPTIONS -lA'
  alias ..='cd ..'
  alias ...='cd ../..'
  alias vi='nvim'
  alias rm='rm -i'
  alias cp='cp -i'
  alias mv='mv -i'
  
  if [[ $- == *i* ]] # execute if in interactive shell
  then
   [[ ! ${BLE_VERSION-} ]] || ble-attach
  fi
  BASHRC
  
  cat <<- 'INITVIM' > /root/.config/nvim/init.vim
  set number
  set expandtab ts=2 sw=2 ai
  set listchars=eol:¬,tab:>·,trail:~,extends:>,precedes:<,space:␣
  set list
  set mouse=
  INITVIM
  
  cat <<- 'MOTDSH' > /etc/profile.d/motd.sh
  #!/usr/bin/env bash
  fastfetch \
    --logo none \
    --structure kernel:os:packages:uptime:loadavg:memory:cpu:disk:shell:localip:publicip
    
  /root/bin/zpool-bar
  
  MOTDSH
  
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

  # complete_requote_threshold compares the quoted length with the escaped one; a single special char loses by one -> discount the two quotes
  function my/patch-requote-threshold {
    local def; def=$(declare -f ble/complete/action/requote-final-insert)
    builtin eval -- "${def/'((${#ret}+threshold<=${#ins}))'/'((${#ret}-2+threshold<=${#ins}))'}"
  }

  # ambiguous completion inserts the common prefix backslash-escaped, requote as an open '...
  function my/requote-common-insert {
    [[ " ${ADVICE_FUNCNAME[*]} " == *' ble/complete/insert-common '* ]] && ((cand_count>1)) || return 0
    local word=${ADVICE_WORDS[3]}
    [[ $word != "$COMPS" && $word == *\\* && $COMPS != *[\\\$\`~=:{]* && $comps_flags != *[SEDI]* ]] || return 0
    local ret simple_flags simple_ibrace count
    ble/syntax:bash/simple-word/reconstruct-incomplete-word "$word" &&
      ble/complete/source/eval-simple-word "$ret" single:count && ((count==1)) || return 0
    local q=\' Q="'\\''"
    ADVICE_WORDS[3]=$q${ret//$q/$Q}
  }
  blehook complete_load!='my/patch-requote-threshold; ble/function#advice before ble/complete/insert my/requote-common-insert'

  # menu-complete (tab-tab cycling) inserts the raw escaped candidate, requote like the final insert
  function my/requote-menu-selection {
    local nsel=${ADVICE_WORDS[1]}
    ((nsel>=0)) && [[ :$bleopt_complete_menu_complete_opts: == *:insert-selection:* ]] || return 0
    local COMP1=${_ble_complete_menu0_comp[0]} COMP2=${_ble_complete_menu0_comp[1]}
    local COMPS=${_ble_complete_menu0_comp[2]} COMPV=${_ble_complete_menu0_comp[3]}
    local comp_type=${_ble_complete_menu0_comp[4]} comps_flags=${_ble_complete_menu0_comp[5]} comps_fixed=${_ble_complete_menu0_comp[6]}
    local "${_ble_complete_cand_varnames[@]/%/=}"
    ble/complete/cand/unpack "${_ble_complete_menu_items[nsel]}"
    local insert=$INSERT insert_flags= suffix=
    ble/complete/action/requote-final-insert
    [[ $insert != "$INSERT" ]] || return 0
    ble-edit/content/replace-limited "$_ble_complete_menu0_beg" "$_ble_edit_ind" "$insert"
    ((_ble_edit_ind=_ble_complete_menu0_beg+${#insert}))
  }
  blehook complete_load!='ble/function#advice after ble/complete/menu-complete.class/onselect my/requote-menu-selection'
  ble-import -d integration/fzf-completion
  ble-import -d integration/fzf-key-bindings
  BLESHRC
  
  cat <<- 'ATUIN' > "/root/.config/atuin/config.toml"
  style = "full"
  enter_accept = true
  records = true
  auto_sync = true
  sync_frequency = 0
  sync_address = "http://10.0.1.6:8888"
  ATUIN
  
  cat <<- 'ZPOOLBAR' > /root/bin/zpool-bar
  #!/usr/bin/env bash
  
  max_usage=90
  bar_width=50
  white="\e[39m"
  green="\e[1;32m"
  red="\e[1;31m"
  dim="\e[2m"
  undim="\e[0m"
  
  printf "\nzpool status:\n"
  zpool status -x | sed -e 's/^/  /'
  
  mapfile -t zpools < <(zpool list -Ho name,cap,size)
  printf "\nzpool usage:\n"
  
  for line in "${zpools[@]}"; do
  
    usage=$(echo "$line" | awk '{print $2}' | sed 's/%//')
    used_width=$((($usage*$bar_width)/100))
  
    if [ "${usage}" -ge "${max_usage}" ]; then
      color=$red
    else
      color=$green
    fi
  
    bar="[${color}"
    for ((i=0; i<$used_width; i++)); do
      bar+="="
    done
  
    bar+="${white}${dim}"
    for ((i=$used_width; i<$bar_width; i++)); do
      bar+="="
    done
    bar+="${undim}]"
  
    echo "${line}" | awk '{ printf("%-30s%+3s used out of %+5s\n", $1, $2, $3); }' | sed -e 's/^/  /'
    echo -e "${bar}" | sed -e 's/^/  /'
  done
  ZPOOLBAR
  
  cat <<- DNFAUTOMATIC > /etc/dnf/automatic.conf
  [commands]
  upgrade_type = default
  random_sleep = 0
  network_online_timeout = 60
  download_updates = yes
  apply_updates = yes
  reboot = never
  reboot_command = "shutdown -r +5 'Rebooting after applying package updates'"
  [emitters]
  emit_via = stdio,email
  [email]
  email_from = root@${hostname}
  email_to = ops@example.com
  email_host = localhost
  [command]
  [command_email]
  [base]
  debuglevel = 2
  DNFAUTOMATIC
  
  mkdir -p /etc/systemd/system/dnf-automatic.timer.d
  
  cat <<- 'DNFAUTOMATICTIMER' > /etc/systemd/system/dnf-automatic.timer.d/override.conf
  [Timer]
  RandomizedDelaySec=28800
  OnCalendar=*-*-* 07:00:00
  DNFAUTOMATICTIMER
  
  cat <<- 'SCREENRC' > /root/.screenrc
  termcapinfo xterm* ti@:te@
  logfile "screenlog_%S.log"
  deflog on
  SCREENRC
  
  echo "command[check_security_updates]=/usr/lib64/nagios/plugins/check_updates --quiet --security-only" >> /etc/nagios/nrpe_local.cfg
  systemctl enable --now dnf-automatic.timer
  chmod +x /root/bin/zpool-bar
  rm -f /root/anaconda-ks.cfg /root/changelog.txt /root/original-ks.cfg /root/postinstall.sh 
  rm -rf /root/workdir/ble.sh
  ```

- enterprise linux (8|9) kubernetes

  ```bash
  #!/usr/bin/env bash
  hostname=$(hostname)
  source /etc/os-release
  MAJOR_VERSION="${VERSION_ID%%.*}"
  if [[ "$MAJOR_VERSION" == "8" ]]; then
      powertools="powertools"
  elif [[ "$MAJOR_VERSION" == "9" ]]; then
      powertools="crb"
  else
      echo "Error: Unsupported AlmaLinux major version ($MAJOR_VERSION)." >&2
      exit 1
  fi
  dnf install --assumeyes --enablerepo="${powertools}" \
    s-nail \
    btop \
    fzf \
    bat \
    neovim \
    bash-completion \
    fastfetch \
    glibc-langpack-en \
    git \
    dnf-automatic \
    nagios-plugins-check-updates \
    perl-Params-Validate \
    perl-Math-Calc-Units \
    perl-Class-Accessor \
    perl-Config-Tiny
  
  mkdir -p /root/.config/nvim
  mkdir -p /root/workdir
  mkdir -p /root/bin
  git clone https://github.com/akinomyoga/ble.sh.git /root/workdir/ble.sh
  cd /root/workdir/ble.sh && make install INSDIR=/usr/local/lib/blesh
  curl --proto '=https' --tlsv1.2 -LsSf https://setup.atuin.sh | sh
  mkdir -p /root/.config/nvim
  grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=1"
  
  cat <<- 'BASH_PROFILE' > /root/.bash_profile
  # exports paths and other variables first
  source "$HOME/.atuin/bin/env"
  export PATH=$PATH:$HOME/bin
  export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
  export KUBE_EDITOR=nvim
  export ATUIN_NOBIND="true"
  export SYSTEMD_EDITOR=nvim
  export LANG=en_US.UTF-8
  export LC_ALL=en_US.UTF-8
  export HISTTIMEFORMAT="%F %T "
  export HISTSIZE="100000"
  export LS_OPTIONS='--color=auto'
  # and then aliases and such
  if [[ -f ~/.bashrc ]]; then
    source ~/.bashrc
  fi
  BASH_PROFILE
  
  cat <<- 'BASHRC' > /root/.bashrc
  if [[ $- == *i* ]] # execute if in interactive shell
  then
    PS1='\[\033[1;37m\][`date +%H:%M:%S`]\[\033[1;36m\][\[\033[1;31m\]\u\[\033[1;33m\]@\[\033[1;32m\]\h:\[\033[1;35m\]\w\[\033[1;36m\]]\[\033[1;31m\]\\$\[\033[0m\] '
    shopt -s extglob
    source -- /usr/local/lib/blesh/ble.sh --attach=none
    eval "$(atuin init bash)"
    bind -x '"\C-q": __atuin_history'
    source /usr/share/fzf/shell/key-bindings.bash
    alias cat='bat --style=plain --paging=never'
    if [[ $PS1 && -f /usr/share/bash-completion/bash_completion ]]
    then
      source /usr/share/bash-completion/bash_completion
    fi
  fi
  
  umask 022
  alias ls='ls $LS_OPTIONS'
  alias ll='ls $LS_OPTIONS -l'
  alias l='ls $LS_OPTIONS -lA'
  alias ..='cd ..'
  alias ...='cd ../..'
  alias vi='nvim'
  alias rm='rm -i'
  alias cp='cp -i'
  alias mv='mv -i'
  
  if [[ $- == *i* ]] # execute if in interactive shell
  then
   [[ ! ${BLE_VERSION-} ]] || ble-attach
  fi
  BASHRC
  
  cat <<- 'INITVIM' > /root/.config/nvim/init.vim
  set number
  set expandtab ts=2 sw=2 ai
  set listchars=eol:¬,tab:>·,trail:~,extends:>,precedes:<,space:␣
  set list
  set mouse=
  INITVIM
  
  cat <<- 'MOTDSH' > /etc/profile.d/motd.sh
  #!/usr/bin/env bash
  fastfetch \
    --logo none \
    --structure kernel:os:packages:uptime:loadavg:memory:cpu:disk:shell:localip:publicip
  MOTDSH
  
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

  # complete_requote_threshold compares the quoted length with the escaped one; a single special char loses by one -> discount the two quotes
  function my/patch-requote-threshold {
    local def; def=$(declare -f ble/complete/action/requote-final-insert)
    builtin eval -- "${def/'((${#ret}+threshold<=${#ins}))'/'((${#ret}-2+threshold<=${#ins}))'}"
  }

  # ambiguous completion inserts the common prefix backslash-escaped, requote as an open '...
  function my/requote-common-insert {
    [[ " ${ADVICE_FUNCNAME[*]} " == *' ble/complete/insert-common '* ]] && ((cand_count>1)) || return 0
    local word=${ADVICE_WORDS[3]}
    [[ $word != "$COMPS" && $word == *\\* && $COMPS != *[\\\$\`~=:{]* && $comps_flags != *[SEDI]* ]] || return 0
    local ret simple_flags simple_ibrace count
    ble/syntax:bash/simple-word/reconstruct-incomplete-word "$word" &&
      ble/complete/source/eval-simple-word "$ret" single:count && ((count==1)) || return 0
    local q=\' Q="'\\''"
    ADVICE_WORDS[3]=$q${ret//$q/$Q}
  }
  blehook complete_load!='my/patch-requote-threshold; ble/function#advice before ble/complete/insert my/requote-common-insert'

  # menu-complete (tab-tab cycling) inserts the raw escaped candidate, requote like the final insert
  function my/requote-menu-selection {
    local nsel=${ADVICE_WORDS[1]}
    ((nsel>=0)) && [[ :$bleopt_complete_menu_complete_opts: == *:insert-selection:* ]] || return 0
    local COMP1=${_ble_complete_menu0_comp[0]} COMP2=${_ble_complete_menu0_comp[1]}
    local COMPS=${_ble_complete_menu0_comp[2]} COMPV=${_ble_complete_menu0_comp[3]}
    local comp_type=${_ble_complete_menu0_comp[4]} comps_flags=${_ble_complete_menu0_comp[5]} comps_fixed=${_ble_complete_menu0_comp[6]}
    local "${_ble_complete_cand_varnames[@]/%/=}"
    ble/complete/cand/unpack "${_ble_complete_menu_items[nsel]}"
    local insert=$INSERT insert_flags= suffix=
    ble/complete/action/requote-final-insert
    [[ $insert != "$INSERT" ]] || return 0
    ble-edit/content/replace-limited "$_ble_complete_menu0_beg" "$_ble_edit_ind" "$insert"
    ((_ble_edit_ind=_ble_complete_menu0_beg+${#insert}))
  }
  blehook complete_load!='ble/function#advice after ble/complete/menu-complete.class/onselect my/requote-menu-selection'
  ble-import -d integration/fzf-completion
  ble-import -d integration/fzf-key-bindings
  BLESHRC
  
  cat <<- 'ATUIN' > "/root/.config/atuin/config.toml"
  style = "full"
  enter_accept = true
  records = true
  auto_sync = true
  sync_frequency = 0
  sync_address = "http://10.0.1.6:8888"
  ATUIN
  
  cat <<- DNFAUTOMATIC > /etc/dnf/automatic.conf
  [commands]
  upgrade_type = security
  random_sleep = 0
  network_online_timeout = 60
  download_updates = yes
  apply_updates = yes
  reboot = never
  reboot_command = "shutdown -r +5 'Rebooting after applying package updates'"
  [emitters]
  emit_via = stdio,email
  [email]
  email_from = root@${hostname}
  email_to = ops@example.com
  email_host = localhost
  [command]
  [command_email]
  [base]
  debuglevel = 2
  DNFAUTOMATIC
  
  mkdir -p /etc/systemd/system/dnf-automatic.timer.d
  
  cat <<- 'DNFAUTOMATICTIMER' > /etc/systemd/system/dnf-automatic.timer.d/override.conf
  [Timer]
  RandomizedDelaySec=28800
  OnCalendar=*-*-* 07:00:00
  DNFAUTOMATICTIMER
  
  cat <<- 'SCREENRC' > /root/.screenrc
  termcapinfo xterm* ti@:te@
  logfile "screenlog_%S.log"
  deflog on
  SCREENRC
  
  echo "command[check_security_updates]=/usr/lib64/nagios/plugins/check_updates --quiet --security-only" >> /etc/nagios/nrpe_local.cfg
  
  systemctl enable --now dnf-automatic.timer
  systemctl enable --now postfix
  echo "sample message" | mail -r "${hostname}" -s "sample mail subject" ops@example.com
  
  rm -f /root/anaconda-ks.cfg /root/changelog.txt /root/original-ks.cfg /root/postinstall.sh
  rm -rf /root/workdir/ble.sh
  ```

- enterprise linux 7 generic

  ```sh
  #!/usr/bin/env bash
  
  sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS* && sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.epel.cloud|g' /etc/yum.repos.d/CentOS*
  yum install neovim bash-completion git -y
  yum install https://github.com/fastfetch-cli/fastfetch/releases/download/1.6.3/fastfetch-1.6.3-Linux.rpm -y
  wget -O bat.zip https://github.com/sharkdp/bat/releases/download/v0.7.1/bat-v0.7.1-x86_64-unknown-linux-musl.tar.gz && tar xzf bat.zip -C /usr/local/ && mv /usr/local/bat-v0.7.1-x86_64-unknown-linux-musl/bat /usr/local/bin/
  mkdir -p /root/workdir
  git clone https://github.com/akinomyoga/ble.sh.git /root/workdir/ble.sh
  cd /root/workdir/ble.sh && make install INSDIR=/usr/local/lib/blesh
  curl --proto '=https' --tlsv1.2 -LsSf https://setup.atuin.sh | sh
  git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf && ~/.fzf/install
  
  cat <<- 'BASH_PROFILE' > /root/.bash_profile
  # exports paths and other variables first
  source "$HOME/.atuin/bin/env"
  export ATUIN_NOBIND="true"
  export PATH=$PATH:$HOME/bin
  export SYSTEMD_EDITOR=nvim
  export LANG=en_US.UTF-8
  export LC_ALL=en_US.UTF-8
  export HISTTIMEFORMAT="%F %T "
  export HISTSIZE="100000"
  export LS_OPTIONS='--color=auto'
  # and then aliases and such
  if [[ -f ~/.bashrc ]]; then
    source ~/.bashrc
  fi
  BASH_PROFILE
  
  cat <<- 'BASHRC' > /root/.bashrc
  if [[ $- == *i* ]] # execute if in interactive shell
  then
    PS1='\[\033[1;37m\][`date +%H:%M:%S`]\[\033[1;36m\][\[\033[1;31m\]\u\[\033[1;33m\]@\[\033[1;32m\]\h:\[\033[1;35m\]\w\[\033[1;36m\]]\[\033[1;31m\]\\$\[\033[0m\] '
    shopt -s extglob
    source -- /usr/local/lib/blesh/ble.sh --attach=none
    eval "$(atuin init bash)"
    bind -x '"\C-q": __atuin_history'
    [[ -f ~/.fzf.bash ]] && source ~/.fzf.bash
    alias cat='bat --style=plain --paging=never'
    if [[ $PS1 && -f /usr/share/bash-completion/bash_completion ]]
    then
      source /usr/share/bash-completion/bash_completion
    fi
  fi 
    
  umask 022
  alias ls='ls $LS_OPTIONS'
  alias ll='ls $LS_OPTIONS -l'
  alias l='ls $LS_OPTIONS -lA'
  alias ..='cd ..'
  alias ...='cd ../..'
  alias vi='nvim'
  alias rm='rm -i'
  alias cp='cp -i'
  alias mv='mv -i'
  
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
  
  cat <<- 'MOTDSH' > /etc/profile.d/motd.sh
  #!/usr/bin/env bash
  fastfetch \
    --logo none \
    --structure kernel:os:packages:uptime:memory:cpu:disk:shell:localip:publicip
  MOTDSH
  
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

  # complete_requote_threshold compares the quoted length with the escaped one; a single special char loses by one -> discount the two quotes
  function my/patch-requote-threshold {
    local def; def=$(declare -f ble/complete/action/requote-final-insert)
    builtin eval -- "${def/'((${#ret}+threshold<=${#ins}))'/'((${#ret}-2+threshold<=${#ins}))'}"
  }

  # ambiguous completion inserts the common prefix backslash-escaped, requote as an open '...
  function my/requote-common-insert {
    [[ " ${ADVICE_FUNCNAME[*]} " == *' ble/complete/insert-common '* ]] && ((cand_count>1)) || return 0
    local word=${ADVICE_WORDS[3]}
    [[ $word != "$COMPS" && $word == *\\* && $COMPS != *[\\\$\`~=:{]* && $comps_flags != *[SEDI]* ]] || return 0
    local ret simple_flags simple_ibrace count
    ble/syntax:bash/simple-word/reconstruct-incomplete-word "$word" &&
      ble/complete/source/eval-simple-word "$ret" single:count && ((count==1)) || return 0
    local q=\' Q="'\\''"
    ADVICE_WORDS[3]=$q${ret//$q/$Q}
  }
  blehook complete_load!='my/patch-requote-threshold; ble/function#advice before ble/complete/insert my/requote-common-insert'

  # menu-complete (tab-tab cycling) inserts the raw escaped candidate, requote like the final insert
  function my/requote-menu-selection {
    local nsel=${ADVICE_WORDS[1]}
    ((nsel>=0)) && [[ :$bleopt_complete_menu_complete_opts: == *:insert-selection:* ]] || return 0
    local COMP1=${_ble_complete_menu0_comp[0]} COMP2=${_ble_complete_menu0_comp[1]}
    local COMPS=${_ble_complete_menu0_comp[2]} COMPV=${_ble_complete_menu0_comp[3]}
    local comp_type=${_ble_complete_menu0_comp[4]} comps_flags=${_ble_complete_menu0_comp[5]} comps_fixed=${_ble_complete_menu0_comp[6]}
    local "${_ble_complete_cand_varnames[@]/%/=}"
    ble/complete/cand/unpack "${_ble_complete_menu_items[nsel]}"
    local insert=$INSERT insert_flags= suffix=
    ble/complete/action/requote-final-insert
    [[ $insert != "$INSERT" ]] || return 0
    ble-edit/content/replace-limited "$_ble_complete_menu0_beg" "$_ble_edit_ind" "$insert"
    ((_ble_edit_ind=_ble_complete_menu0_beg+${#insert}))
  }
  blehook complete_load!='ble/function#advice after ble/complete/menu-complete.class/onselect my/requote-menu-selection'
  ble-import -d integration/fzf-completion
  ble-import -d integration/fzf-key-bindings
  BLESHRC
  
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
  
  rm -f /root/anaconda-ks.cfg /root/changelog.txt /root/original-ks.cfg /root/postinstall.sh /root/bat.zip
  rm -rf /root/workdir/ble.sh
  ```
