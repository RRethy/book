# Keeping Neovim and Kitty Terminal Colorschemes Consistent and Persistent

Whether it's to give my editor a new coat of paint or to make it easier on the eyes for 3am programming, I change its colours all the time. In editors such as Visual Studio Code or Atom, this is a simple matter of choosing a theme from a fuzzy matched drop-down menu. However, in Neovim we have to edit the `init.lua` and restart the editor (or source the `init.lua`). On top of this, we then need to update our terminal's colours to match (or at least I like to keep them consistent). This is tedious and nowhere near as clean as VSCode.

This tutorial will create a similar experience to VSCode where you can select a colorscheme using a dropdown fuzzy finder, and this colorscheme will remain consistent in Neovim and Kitty even if you close them and reopen.

**Note**: We will be using [base16](https://github.com/chriskempson/base16) colours but you can swap these out for something else.

We will use a file, `$XDG_CONFIG_HOME/.base16_theme`, as the single source of truth for our current colours. It will contain a single line with the name of our current base16 colours. For example,

```
base16-gruvbox-dark-hard
```

## Kitty (Terminal) Setup

**Note**: These steps only work in [kitty](https://sw.kovidgoyal.net/kitty/).

1. Download [base16-kitty](https://github.com/kdrag0n/base16-kitty) to `$HOME`
2. In a shell startup script (e.g. `~/.zshrc`), add `eval "kitty @ set-colors -c $HOME/base16-kitty/colors/$(cat $XDG_CONFIG_HOME/.base16_theme).conf"`. This will set the colours for the current terminal window and for newly created terminal windows.

## Neovim Setup

1. Install [nvim-base16](https://github.com/rrethy/nvim-base16) plugin. You can use a different collection of colorschemes but this will match the themes in [base16-kitty](https://github.com/kdrag0n/base16-kitty) we used above.

2. Create a function to update the colours for our current terminal window and current instance of Neovim. We'll also need to update our `$XDG_CONFIG_HOME/.base16_theme`.

```lua
-- this is our single source of truth created above
local base16_theme_fname = vim.fn.expand(vim.env.XDG_CONFIG_HOME..'/.base16_theme')
-- this function is the only way we should be setting our colorscheme
local function set_colorscheme(name)
    -- write our colorscheme back to our single source of truth
    vim.fn.writefile({name}, base16_theme_fname)
    -- set Neovim's colorscheme
    vim.cmd('colorscheme '..name)
    -- execute `kitty @ set-colors -c <color>` to change terminal window's
    -- colors and newly created terminal windows colors
    vim.loop.spawn('kitty', {
        args = {
            '@',
            'set-colors',
            '-c',
            string.format(vim.env.HOME..'/base16-kitty/colors/%s.conf', name)
        }
    }, nil)
end
```

3. Read `$XDG_CONFIG_HOME/.base16_theme` to determine our current colours to use for Neovim.

```lua
set_colorscheme(vim.fn.readfile(base16_theme_fname)[1])
```

4. Install [telescope.nvim](https://github.com/nvim-telescope/telescope.nvim). Other fuzzy pickers will work, but this code snippet is for this plugin.

5. Override the `colorschemes` picker in `telescope.nvim` to update our terminal colours too

```lua
nvim.nnoremap('<leader>c', function()
    -- get our base16 colorschemes
    local colors = vim.fn.getcompletion('base16', 'color')
    -- we're trying to mimic VSCode so we'll use dropdown theme
    local theme = require('telescope.themes').get_dropdown()
    -- create our picker
    require('telescope.pickers').new(theme, {
        prompt = 'Change Base16 Colorscheme',
        finder = require('telescope.finders').new_table {
            results = colors
        },
        sorter = require('telescope.config').values.generic_sorter(theme),
        attach_mappings = function(bufnr)
            -- change the colors upon selection
            telescope_actions.select_default:replace(function()
                set_colorscheme(action_state.get_selected_entry().value)
                telescope_actions.close(bufnr)
            end)
            telescope_action_set.shift_selection:enhance({
                -- change the colors upon scrolling
                post = function()
                    set_colorscheme(action_state.get_selected_entry().value)
                end
            })
            return true
        end
    }):find()
end)
```

## Try it out

Open up Neovim, and hit `<leader>c` (or whatever mapping you used) and search for a colorscheme. As you scroll through the available colorschemes Neovim should update and Kitty should remain consistant. If you close then open Neovim, the colorscheme will persist with manually changing anything else. Not only that, but you can close and open Kitty and both Kitty and Neovim will maintain the previously selected colorscheme.
