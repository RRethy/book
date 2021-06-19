# Write Your Own Plugin Manager With A Focus On Ergonomics

*Note: This is written with Neovim in mind but the same concepts can be applied to Vim*

I've been pretty dissatisfied with the state of plugin managers in Neovim. Almost all of them follow a similar flow of editing your `init.lua`/`init.vim`, running an install command, and restarting Neovim to have the plugin loaded and ready to use. If we compare that with VSCode, we click a button to install the extension, and it usually *just works* with the occasional need to restart the editor. It's a lot more streamlined in VSCode and not having to restart the editor or manually edit config files is much smoother. What's worse is this is all functionality that can be mimicked in Neovim with a clever use of `:h packages`.

At the end of this article we'll have a small plugin manager (~130 lines of Lua) that will let you install the plugin with a simple `:PackAdd https://github.com/RRethy/vim-illuminate` which will make it instantly available without the need for a restart and you won't need to touch your dotfiles. Removing the plugin will be as easy as deleting a line in the manifest (we'll get to this later).

*Note: This code will not be put into it's own repo but you can still add it to your own dotfiles for use (I currently use it). I'm hoping that this style of plugin manager pushes new plugin managers to create a more ergonomic experience.*

## High-Level Overview

As with most plugin managers nowadays, we will build on top of the `:help packages` feature. However, unlike other managers, we will not use the `start/` directory to store plugins since those are all sourced at startup, instead we will download all plugins into `opt/` and execute a `:packadd <plugin>` to load it dynamically without restarting Neovim. We will also have a separate file that we'll call a pack manifest which is where our plugins will be listed, this manifest will be read at startup and each plugin will be `:packadd!` so it gets sourced, this has the added benefit that deleting a plugin is just deleting a line in the manifest.

*Note: Not all plugins can be dynamically loaded, just most of them.*

## Picking a Name

*TLDR: backpack.lua*

Picking a name is always tough, should we aim for descriptiveness (like `nvim-treesitter-textobjects`) or attempt to confuse the user (like `wildfire.vim` or `vim-hexokinase`), that is the question. Personally, I prefer names with personalities which are at least somewhat indicative of their functionality. A tried a true method for this would be to take a word that is directly related (like `pack` since we're using `:h packages`) and search for superstrings of that word which also make sense (https://www.thefreedictionary.com/words-containing-pack). We'll use `backpack.lua` since it sounds good enough.

## Entry Point

Let's create a file `lua/backpack.lua` (in our dotfiles) which will hold all of our code.

The directory for our plugins will be `vim.fn.stdpath('data')..'/site/pack/backpack/opt/'` which should be recognized by `:h 'packpath'` but if it isn't then add a `vim.opt.packpath:append('~/.local/share/nvim/site')` to your `init.lua`.

As mentioned above we'll have a manifest to specify what plugins we are using, we'll place this at `vim.fn.stdpath('config')..'/packmanifest.lua'` inside our dotfiles. Our manifest is a Lua file that we will `dofile()` and looks like this:

```lua
use { 'RRethy/nvim-treesitter-textsubjects' }
use {
    'RRethy/vim-hexokinase',
    post_update = { 'make' }
}
use { 'tpope/vim-fugitive' }
```

`use` will be a global we define specifically for use in the manifest to declare a plugin so it gets loaded and we track it in case it should be updated later on.

We'll also want to declare 3 commands for later use: `:PackAdd` to add a plugin, `:PackUpdate` to update our plugins, `:PackEdit` to view/edit the manifest.

Putting this into code looks like the following:

```lua
local M = {}

local opt = vim.fn.stdpath('data')..'/site/pack/backpack/opt/'
local manifest = vim.fn.stdpath('config')..'/packmanifest.lua'

local GITHUB_USERNAME = '<your github username>'

local function to_git_url(author, plugin)
    if author == GITHUB_USERNAME then
        return string.format('git@github.com:%s/%s.git', author, plugin)
    else
        return string.format('https://github.com:%s/%s.git', author, plugin)
    end
end

function M.setup()
    vim.fn.mkdir(opt, 'p') -- make sure opt exists

    M.plugins = {}
    -- 'use' will be define only for use in the manifest
    _G.use = function(opts)
        local _, _, author, plugin = string.find(opts[1], '^([^ /]+)/([^ /]+)$')
        -- track the plugin so it can be updated later with :PackUpdate
        table.insert(M.plugins, {
            plugin = plugin,
            author = author,
            post_update = opts.post_update,
        })
        -- adds the plugin to the end of :help 'runtimepath'
        -- this will be what makes the plugin sourced
        -- NOTE: :packadd! is not the same as :packadd
        if vim.fn.isdirectory(opt..'/'..plugin) ~= 0 then
            vim.cmd('packadd! '..plugin)
        else
            git_clone(plugin, to_git_url(author, plugin), function()
                vim.cmd('packadd! '..plugin)
            end)
        end
    end
    if vim.fn.filereadable(manifest) ~= 0 then
        dofile(manifest)
    end
    _G.use = nil

    -- these functions will be defined later, you can add a --bar too if you want to chain command usage
    vim.cmd [[ command! -nargs=1 PackAdd lua require('rrethy.backpack').pack_add(<f-args>) ]]
    vim.cmd [[ command! PackUpdate lua require('rrethy.backpack').pack_update() ]]
    vim.cmd [[ command! PackEdit lua require('rrethy.backpack').pack_edit() ]]
end
```

Then our `init.lua` will have:

```lua
require('backpack').setup()
```

## :PackAdd

Before we add code to install a plugin, we'll need a few helpers to pull Git repos, clone Git repos, and parse a GitHub URL.

We'll start with parsing a GitHub URL. I'm a big fan of just copy pasting the URL (`<cmd>l<cmd>c` on OSX) after a `:PackAdd `, it's quite ergonomic compared to looking for installation instructions or copying the exact part of the URL.

```lua
local function parse_url(url)
    -- regex capture the username and plugin name from the url
    local username, plugin = string.match(url, '^https://github.com/([^/]+)/([^/]+)$')
    if not username or not plugin then
        -- failed to parse, we can spit an error out here
        return
    end

    local git_url
    if username == GITHUB_USERNAME then
        -- a nicety for plugins that you wrote, prefer ssh over https
        git_url = string.format('git@github.com:%s/%s.git', username, plugin)
    else
        git_url = string.format('https://github.com/%s/%s.git', username, plugin)
    end

    return git_url, username, plugin
end
```

Now we'll want functions to clone and pull Git repos which will be used by `:PackAdd` and `:PackUpdate`

```lua
local function git_pull(name, on_success)
    local dir = opt..name
    -- get the branch name, there might be a better way to do this
    local branch = vim.fn.system("git -C "..dir.." branch --show-current | tr -d '\n'")
    -- use Luv to execute an async `git pull` with a shallow fetch
    vim.loop.spawn('git', {
        args = { 'pull', 'origin', branch, '--update-shallow', '--ff-only', '--progress', '--rebase=false' },
        cwd = dir,
    }, vim.schedule_wrap(function(code)
            if code == 0 then
                on_success(name)
            else
                echoerr(name..' pulled unsuccessfully')
            end
        end))
end

local function git_clone(name, git_url, on_success)
    -- use Luv to execute an async `git clone` with a shallow clone
    vim.loop.spawn('git', {
        args = { 'clone', '--depth=1', git_url },
        cwd = opt,
    }, vim.schedule_wrap(function(code)
            if code == 0 then
                on_success(name)
            else
                echoerr(name..' cloned unsuccessfully')
            end
        end))
end
```

Now we can write our actual `:PackAdd` functionality:

```lua
function M.pack_add(url)
    local git_url, author, plugin = parse_url(url)
    if not git_url then
        -- failed to parse url
        return
    end

    -- track the plugin in case of a :PackUpdate later
    table.insert(M.plugins, {
        plugin = plugin,
        author = author,
    })
    local on_success = function()
        -- if successful, try loading the plugin dynamically without restarting Neovim
        vim.cmd('packadd '..plugin)
    end
    if vim.fn.isdirectory(opt..plugin) ~= 0 then
        git_pull(plugin, on_success)
    else
        git_clone(plugin, git_url, on_success)
    end

    -- automatically add the plugin data to our manifest
    vim.fn.system(string.format('echo "use { \'%s/%s\' }" >> %s', author, plugin, manifest))
end
```

## :PackUpdate

Since we were tracking our plugins in `M.plugins`, we can now just clone/pull each of them.

```lua
function M.pack_update()
    for _, data in ipairs(M.plugins) do
        local on_success = function(plugin)
            vim.cmd('packadd '..plugin)
        end
        if vim.fn.isdirectory(opt..data.plugin) ~= 0 then
            git_pull(data.plugin, on_success)
        else
            git_clone(data.plugin, git_clone, on_success)
        end
    end
end
```

## :PackEdit

Since the manifest is just a Lua file, we can just open it up in a new tab. Closing the manifest is as simple as `:bw`. While this may seem trivial, IMO it's enough.

```lua
function M.pack_edit()
    vim.cmd('tabnew')
    vim.cmd('edit '..manifest)
end
```

## Post-Update Hooks

We were also tracking post-update hooks which is the only additional feature we'll be adding (see below for other feature that were intentionally omitted). This can take the form of a lua function callback or a table representing shell commands to run in the root of the plugin.

We'll add this in `M.pack_update` after we dynamically reload the plugin in the `on_success` callback.

```lua
if data.post_update then
    -- plugin root directory
    local dir = opt..'/'..plugin
    if type(data.post_update) == 'function' then
        -- execute the function and pass it the plugin dir
        data.post_update(dir)
    elseif type(data.post_update) == 'table' then
        -- use Luv to run the shell command in the plugin dir
        vim.loop.spawn(data.post_update[1], { args = data.post_update.args, cwd = dir },
            vim.schedule_wrap(function(code)
                if code ~= 0 then
                    vim.api.nvim_err_writeln(string.format('Failed to run %s', vim.inspect(data.post_update)))
                end
            end))
    end
end
```

### What We Left Out (And Why)

1. Lazy loading on filetype
    - This frustrates me to no end, this feature is redundant!!! I've seen far too many `Plug 'vim-ruby/vim-ruby', { 'for': 'ruby' }`, just look at the source code for the plugin, it already lazily loads for `ruby` or `eruby` filetypes. In fact, most plugins lazy load most of their code until it's actually used. The few plugins which don't are typically quite old or if the plugin author doesn't know about `ftplugin/` or `autoload/` (probably should look for another plugin in that case). On top of this, I don't think people know that this doesn't restrict the plugin to that filetype, once it gets loaded, it's there for all filetypes.
2. Lazy loading on command
    - Same reason as above, most plugins are already lazy loaded. Before you lazy load on `:Foo`, take a look at it's definition to see if it's just calling out to an autoloaded function which hasn't been sourced, if so, then it's already lazy loaded. If a plugin is doing massive amounts of work at startup then it might be time to look for a better written plugin.
3. Lua Rocks support
    - This would be nice to have but I hope this becomes part of Neovim rather than forcing plugin managers to add support for it.
4. Rigorous Error Handling
    - I trimmed it off to reduce the complexity and size, simply writing errors to a log file is an easy way to add it back.
