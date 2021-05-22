# Keeping Neovim and Kitty Terminal Colorschemes Consistent and Persistent

Whether it's to give it a new coat of paint or to make it easier on the eyes for 3am programming, I change my editor colours all the time. In editors likes Visual Studio Code or Atom, this is a simple matter of choosing one from a fuzzy matched drop-down menu. However, in Neovim we have to edit the `init.lua` so it sets a different colorscheme then restart the editor (or manually choose the same colorscheme with `:colorscheme <name>`). On top of this, we then need to update our terminal's colours to match (or at least I do). Let's see how close we can get to the VSCode experience.

**Note**: We will be using [base16](https://github.com/chriskempson/base16) colours, but this works equally well without them.

## Single Source of Truth

We will use a file `~/.config/.base16_theme` as the single source of truth for our current colours. It will contain a single line with the name of our current base16 colours.

## Kitty (Terminal) Setup

**Note**: These steps only work in [kitty](https://sw.kovidgoyal.net/kitty/) but a similar procedure could be used for other editors.

1. Download [base16-kitty](https://github.com/kdrag0n/base16-kitty) (This tutorial assumes `~/.config/kitty/` as the download location)
2. In a shell startup script (e.g. `~/.zshrc`), add `eval "kitty @ set-colors -c $HOME/.config/kitty/base16-kitty/colors/$(cat $XDG_CONFIG_HOME/.base16_theme).conf"`. This will set the colours for the current terminal window and for newly created terminal windows.

## Neovim

We will be adding the following code snippets to our `init.lua`.

1. Install [nvim-base16](https://github.com/rrethy/nvim-base16) plugin. It's optional but it has all the colorschemes we need.

2. Read `~/.config/.base16_theme` to determine our current colours to use for Neovim.

```lua
local base16_theme_fname = vim.fn.expand('~/.config/.base16_theme')
vim.cmd('colorscheme '..vim.fn.readfile(base16_theme_fname)[1])
```

3. Create a function to update the colours for our current terminal window.

```lua
function set_kitty_colors(name)
    vim.loop.spawn('kitty', {
        args = {
            '@',
            '--to',
            vim.env.KITTY_LISTEN_ON,
            'set-colors',
            '-c'
            string.format('~/.config/kitty/base16-kitty/colors/%s.conf', name)
        }
    }, nil)
end
```

4. Install [telescope.nvim](https://github.com/nvim-telescope/telescope.nvim). It's also optional since other fuzzy finders can be used, but the code snippets will be specific to this plugin.

5. Override the `colorschemes` picker in `telescope.nvim` to update our terminal colours too

```lua
nvim.nnoremap('<leader>c', function()
    return telescope_builtin.colorscheme(telescope_themes.get_dropdown({
        prompt_title = 'Change Colorscheme',
        attach_mappings = function(bufnr)
            telescope_actions.select_default:replace(function()
                local name = action_state.get_selected_entry().value
                vim.fn.writefile({name}, base16_theme_fname)
                set_colorscheme(name)
                telescope_actions.close(bufnr)
            end)
            telescope_action_set.shift_selection:enhance({
                post = function()
                    local name = action_state.get_selected_entry().value
                    vim.fn.writefile({name}, base16_theme_fname)
                    set_colorscheme(name)
                end
            })
        return true
    end}))
end)
```

## Conclusion

Try it out with `<leader>c`, it'll update your Neovim colours and persist this change as your restart either Neovim or your Terminal. It's about as hands off as your can get.
