---
title: cheat sheet/ble.sh
---

- version check -- 0.3.x (distro packages, 2019) has no requote and writes history only on clean bash exit

  ```sh
  grep -m1 _ble_init_version= /usr/share/blesh/ble.sh
  # arch: sudo pacman -Rdd blesh && paru -S blesh-git
  # source: git clone https://github.com/akinomyoga/ble.sh.git
  #         cd ble.sh && make install INSDIR=/usr/local/lib/blesh
  ```

- history, crash-safe -- in `.bashrc`, NOT `bleopt history_share=1`

  ```sh
  export HISTFILESIZE="$HISTSIZE"
  shopt -s histappend
  PROMPT_COMMAND='history -a'
  ```

- why not history_share (0.4.0-devel4) -- `ble/builtin/history/.initialize`

  ```sh
  rskip = wc -l "$HISTFILE"                        # LINE count
  ((max && max-min+1 < rskip && (rskip=max-min+1))) # ENTRY count
  # HISTTIMEFORMAT set -> 2 lines per entry -> rskip halves every start
  # -> second half of HISTFILE re-read as new, appended back, compounds per shell
  # 4 days, a few screen windows: 11,608 unique entries -> 1,000,001 (34 MB)
  ```

- histfile already bloated -- kill the old shells FIRST, they rewrite it from memory on the next prompt

  ```sh
  pgrep -a -u "$USER" -x bash    # anything started before the config change
  awk '/^#[0-9]+$/{t=$0;next}{k=t"\t"$0; if(!(k in s)){s[k]=1; if(t)print t; print}}' \
    ~/.bash_history > /tmp/h && mv ~/.bash_history ~/.bash_history.bak && mv /tmp/h ~/.bash_history
  ```

- tab completion quotes with `'...'` instead of backslashes -- needs 0.4, `complete_requote_threshold=0` is the default but only covers the final unique insert. full blerc with the 4 hooks: [comfy-env/files/blerc](https://github.com/sandravwc/comfy-env/blob/master/files/blerc)

  ```sh
  # the three it does NOT cover on its own:
  # rsync/scp   -> bash-completion pre-escapes scp-style, ble.sh quotes again
  # ambiguous   -> common prefix inserted backslash-escaped
  # tab-tab     -> menu cycling inserts the raw escaped candidate
  # plus: threshold compares quoted vs escaped length, one special char loses by 1
  ```

- advice hooks, gotchas

  ```sh
  # completion lib loads lazily -> attach via blehook complete_load, not at rc time
  blehook complete_load!='ble/function#advice after <func> <mine>'
  # inside an advice: ADVICE_WORDS[0] is the function name, args start at [1]
  # fzf owns rsync/scp completion (_fzf_path_completion) and calls the original
  # inside itself -> an after-advice fires twice, guard on $COMP_LINE:$COMP_POINT
  ```

- extglob -- operator before the parens, `shopt -s extglob`

  ```sh
  ?(foo)  # regex (foo)?
  *(foo)  # (foo)*
  +(foo)  # (foo)+
  @(a|b)  # (a|b)
  !(foo)  # negation
  ```
