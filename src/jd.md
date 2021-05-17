# A few lines of shell script to replace/improve autojump/z.lua

TODO

```zsh
mkdir -p "$XDG_DATA_HOME/zshrc"
JD_DATA_DIR="$XDG_DATA_HOME/zshrc/chpwd.txt"
touch $JD_DATA_DIR
local tmp=$(mktemp)
cat $JD_DATA_DIR | while read dir ; do [[ -d $dir ]] && echo $dir ; done > $tmp
cat $tmp > $JD_DATA_DIR
chpwd_functions+=(on_chpwd)
function on_chpwd {
    local tmp=$(mktemp)
    { echo $PWD ; cat $JD_DATA_DIR } | sort | uniq 1> $tmp
    cat $tmp > $JD_DATA_DIR
}
function fzy_jd {
    if [[ ! -z $BUFFER ]]; then
        BUFFER=$BUFFER"jd "
        zle end-of-line
        return 0
    fi
    local dir=$({ echo $HOME ; cat $JD_DATA_DIR } | fzy)
    if [[ -z $dir ]]; then
        zle redisplay
        return 0
    fi
    zle push-line
    BUFFER="cd $dir"
    zle accept-line
    local ret=$?
    zle reset-prompt
    return $ret
}
zle -N fzy_jd
bindkey 'jd ' fzy_jd
alias jd=cd
```
