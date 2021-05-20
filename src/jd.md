# A small ZSH script to replace autojump

Add this to a `~/.zshrc` and ensure you have [fzy](https://github.com/jhawthorn/fzy) installed.

When you type `jd ` (`jd` followed by a space without pressing enter), fzy will pop open and let you change your `$PWD` to a previously visited directory. For details, just read the comments in the code snippet.

**Note**: To change the keybinding, modify the `$JUMPDIR_KEYBIND` variable.

```zsh
JUMPDIR_KEYBIND='jd '
# Setup some data a data file to store visited directories
mkdir -p "$XDG_DATA_HOME/zshrc"
JD_DATA_DIR="$XDG_DATA_HOME/zshrc/chpwd.txt"
touch $JD_DATA_DIR
local tmp=$(mktemp)
cat $JD_DATA_DIR | while read dir ; do [[ -d $dir ]] && echo $dir ; done > $tmp
cat $tmp > $JD_DATA_DIR
# Track visited directories
chpwd_functions+=(on_chpwd)
function on_chpwd {
    local tmp=$(mktemp)
    { echo $PWD ; cat $JD_DATA_DIR } | sort | uniq 1> $tmp
    cat $tmp > $JD_DATA_DIR
}
# zle widget function
function fzy_jd {
    # check if `jd ` was triggered in the middle of another command
    # e.g. $ aaaaaaajd 
    # If so, we manually input the `jd `
    if [[ ! -z $BUFFER ]]; then
        # Append `jd ` to the prompt
        BUFFER=$BUFFER$JUMPDIR_KEYBIND
        # move the cursor to the end of the line
        zle end-of-line
        return 0
    fi
    # ask the user to select a directory to jump to
    local dir=$({ echo $HOME ; cat $JD_DATA_DIR } | fzy)
    if [[ -z $dir ]]; then
        # no directory was selected, reset the prompt to what it was before
        zle reset-prompt
        return 0
    fi
    # Setup the command to change the directory
    BUFFER="cd $dir"
    # Accepts the cd we setup above
    zle accept-line
    local ret=$?
    # force the prompt to redraw to mimic what would occur with a normal cd
    zle reset-prompt
    return $ret
}
# define the new widget function
zle -N fzy_jd
# bind the widget function to `jd `
bindkey $JUMPDIR_KEYBIND fzy_jd
# a nicety so that executing just jd will mimic the behaviour of just executing
# cd, that is, change the pwd to $HOME
eval "alias $(echo $JUMPDIR_KEYBIND|xargs)=cd"
```
