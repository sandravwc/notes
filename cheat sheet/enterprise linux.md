---
title: cheat sheet/enterprise linux
---

- [[cheat sheet/enterprise linux/comfy script]]
- [[cheat sheet/enterprise linux/install atuin server]]
- add vault repo

  ```sh
  sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS* && sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.epel.cloud|g' /etc/yum.repos.d/CentOS*
  ```

- fzf and bat on centos7

  ```sh
  git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf && ~/.fzf/install
  wget -O bat.zip https://github.com/sharkdp/bat/releases/download/v0.7.1/bat-v0.7.1-x86_64-unknown-linux-musl.tar.gz && tar xzf bat.zip -C /usr/local/ && mv /usr/local/bat-v0.7.1-x86_64-unknown-linux-musl/bat /usr/local/bin/
  ```

- comfy

  ```sh
  dnf install fzf bat neovim bash-completion fastfetch glibc-langpack-en -y
  ```

- .bash_profile

  ```sh
  # exports paths and other variables first
  export PATH=$PATH:$HOME/bin
  export SYSTEMD_EDITOR=nvim
  export LANG=en_US.UTF-8
  export LC_ALL=en_US.UTF-8
  export HISTTIMEFORMAT="%F %T "
  export HISTSIZE="100000"
  shopt -s histappend
  PROMPT_COMMAND='history -a'
  export LS_OPTIONS='--color=auto'
  # and then aliases and such
  if [[ -f ~/.bashrc ]]; then
    source ~/.bashrc
  fi
  ```

- bashrc

  ```sh
  tbd
  ```

- centos7 bash-completion

  ```sh
  [[ $PS1 && -f /usr/share/bash-completion/bash_completion ]] && \
      . /usr/share/bash-completion/bash_completion
  ```

- .config/nvim/init.vim

  ```vim
  set number
  set expandtab ts=2 sw=2 ai
  set listchars=eol:¬,tab:>·,trail:~,extends:>,precedes:<,space:␣
  set list
  set mouse=
  ```

- new ssh host keys

  ```sh
  rm -f /etc/ssh/ssh_host_* && ssh-keygen -A
  ```

- bash line editor

  ```sh
  git clone https://github.com/akinomyoga/ble.sh.git
  cd ble.sh && make install INSDIR=/usr/local/lib/blesh
  
  add to .bashrc:
  [[ $- == *i* ]] && source -- /usr/local/lib/blesh/ble.sh --attach=none
  [[ ! ${BLE_VERSION-} ]] || ble-attach
  
  .blerc:
  ble-bind -f 'M-B' 'backward-cword'
  ble-bind -f 'M-F' 'forward-cword'
  bleopt complete_auto_complete=
  bleopt complete_auto_history=
  bleopt complete_ambiguous=
  bleopt prompt_eol_mark=''
  bleopt complete_menu_filter=
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
  ```

- /etc/profile.d/motd.sh

  ```sh
  #!/usr/bin/env bash
  fastfetch \
    --logo none \
    --structure kernel:os:Packages:uptime:memory:Shell:LocalIp:PublicIp
  ```

- resize ext4 fs

  ```sh
  growpart /dev/sda 3
  resize2fs /dev/sda3
  ```

- fix fucked locale

  ```sh
  dnf install glibc-langpack -en
  ```

- check tls certificate

  ```sh
  openssl x509 -noout -text -in cert.crt
  openssl x509 -noout -dates -in cert.crt
  ```
