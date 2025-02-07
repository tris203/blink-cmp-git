# blink-cmp-git

Git source for [blink.cmp](https://github.com/Saghen/blink.cmp)
completion plugin. This makes it possible to query pull requests, issues,
and users from GitHub. This is very useful when you are writing a commit with `nvim`.

Use `#` to search for issues and pull requests:

![blink-cmp-git pr and issues](./images/demo-prs-issues.png)

Use `:` to search for commits:

![blink-cmp-git commits](./images/demo-commits.png)

Use `@` to search for users:

![blink-cmp-git pr and issues](./images/demo-users.png)

## Requirements

`gh` is required for the default configuration.

## Installation

Add the plugin to your packer managers, and make sure it is loaded before `blink.cmp`.

### `lazy.nvim`

```lua
{
    'saghen/blink.cmp',
    dependencies = {
        {
            'Kaiser-Yang/blink-cmp-git',
            dependencies = { 'nvim-lua/plenary.nvim' }
        }
        -- ... other dependencies
    },
    opts = {
        sources = {
            -- add 'git' to the list
            default = { 'git', 'dictionary', 'lsp', 'path', 'luasnip', 'buffer' },
            git = {
                opts = {
                    -- options for the blink-cmp-git
                },
            },
        }
    }
}
```

## Quick Start

```lua
git = {
    -- Because we use filetype to decide whether or not to show the items,
    -- we can make the score higher
    score_offset = 100,
    module = 'blink-cmp-git',
    name = 'Git',
    -- enabled this source at the beginning to make it possible to pre-cache
    -- at very beginning
    enabled = true,
    -- only show this source when filetype is gitcommit or markdown
    should_show_items = function()
        return vim.o.filetype == 'gitcommit' or vim.o.filetype == 'markdown'
    end,
    --- @module 'blink-cmp-git'
    --- @type blink-cmp-git.Options
    opts = {
        commit = {
            -- You may want to custom when it should be enabled
            -- The default will enable this when `cwd` is in a git repository
            -- enable = function() end
            -- You may want to change the triggers
            -- triggers = { ':' },
        }
        git_centers = {
            git_hub = {
                -- Those below have the same fields with `commit`
                -- issues = {
                -- },
                -- pull_request = {
                -- },
                -- mention = {
                -- }
            }
        }
    }
},
```

The configuration above will enable the `blink-cmp-git` for `blink.cmp` and show the items
when the filetype is `gitcommit` or `markdown`. By default, `blink-cmp-git` will pre-cache
everything when it is created. To enable `blink-cmp-git` all the time makes it possible to
pre-cache when you enter insert mode or other mode you can input
(`blink.cmp` will create sources when you can input something).

## Reload Cache

There are many cases will make the cache out of date. For example,
if your `cwd` is in a repository, later you switch your `cwd` to another repository, the cache
will use the first repository's result. To solve this problem, there is a command to
reload the cache: `BlinkCmpGitReloadCache`. This command will clear all the cache and if
`use_items_pre_cache` is enabled (default to `true`), it will pre-cache again.

You can bind the command to a key or create a vim autocommand to reload the cache when your
`cwd` changes.

> [!NOTE]
>
> The command will be available only when the `blink-cmp-git` source is created. Usually,
> the source will be created when it is enabled and you are in some mode you can input.

## Default Configuration

See [default.lua](./lua/blink-cmp-git/default.lua).

## FAQs

### How to custom the completion items?

Because all features have same fields, I'll use `commit` as an example.

The `blink-cmp-git` will first run command from `get_command` and `get_command_args`. The standout
of the command will be passed to `separate_output`. So if you want to custom the completion items,
you should be aware of what the output of your command looks like.

The default `get_command` and `get_command_args` for `commit`:

```lua
get_command = 'git',
get_command_args = {
    '--no-pager',
    'log',
    '--pretty=fuller',
    '--decorate=no',
},
```

This will give you the output like:

```gitcommit
commit 0216336d8ff00d7b8c9304b23bcca31cbfcdf2c8
Author:     Kaiser-Yang <624626089@qq.com>
AuthorDate: Sun Jan 12 14:40:38 2025 +0800
Commit:     Kaiser-Yang <624626089@qq.com>
CommitDate: Sun Jan 12 14:43:15 2025 +0800

    Cache empty documentations

commit 90e2fd0f5ae6e4de00eab63f5cb99f850e0ffa56
Author:     Kaiser-Yang <624626089@qq.com>
AuthorDate: Sun Jan 12 14:27:39 2025 +0800
Commit:     Kaiser-Yang <624626089@qq.com>
CommitDate: Sun Jan 12 14:27:39 2025 +0800

    Improve the experience of pre cache
...
```

The default `separate_output` for `commit`:

```lua
separate_output = function(output)
    local lines = vim.split(output, '\n')
    local i = 1
    local commits = {}
    -- Those below separate the output to a list of commits
    -- I've tried use regex to match the commit, but there always were missing some commits
    while i < #lines do
        local j = i + 1
        while j < #lines do
            if lines[j]:match('^commit ') then
                j = j - 1
                break
            end
            j = j + 1
        end
        commits[#commits + 1] = table.concat(lines, '\n', i, j)
        i = j + 1
    end
    local items = {}
    ---@diagnostic disable-next-line: redefined-local
    for i = 1, #commits do
        --- @type string
        local commit = commits[i]
        items[i] = {
            -- label is what to show in the completion menu
            -- the fist 7 characters of the hash and the subject of the commit
            label =
                commit:match('commit ([^\n]*)'):sub(1, 7)
                ..
                ' '
                ..
                commit:match('\n\n%s*([^\n]*)'),
            -- insert_text is what to insert when you select or accept the item
            insert_text = commit:match('^commit ([^\n]*)'):sub(1, 7) .. ' ',
            -- this can be a `DocumentationCommand` or a string
            -- set this to nil if you don't want to show the documentation
            -- use the whole commit as the documentation
            documentation = commit
            -- documentation = {
            --     -- the command to get the documentation
            --     get_command = '',
            --     get_command_args = {}
            --     -- how to resolve the output
            --     resolve_documentation = function(output) return output end
            -- }
            -- documentation = nil
        }
    end
    return items
end,
```

## Performance

Once `async` is enabled, the completion will has no effect to your other operations.
How long it will take to show results depends on the network speed and the response time
of the git center. But, don't worries, once you enable `use_items_cache`, the items will be
cached when you first trigger the completion by inputting `@`, `#`, or `:`
(You can DIY the triggers). Furthermore, once you enable `use_items_pre_cache`, when the
source is created, it will pre-cache all the items. For the documentation of `mention` feature,
it will be cached when you hover on one item.

## Version Introduction

The release versions are something like `major.minor.patch`. When one of these numbers is increased:

* `patch`: bugs are fixed or docs are added. This will not break the compatibility.
* `minor`: compatible features are added. This may cause some configurations `deprecated`, but
not break the compatibility.
* `major`: incompatible features are added. All the `deprecated` configurations will be removed.
This will break the compatibility.

## Acknowledgment

Nice and fast completion plugin: [blink.cmp](https://github.com/Saghen/blink.cmp).

Inspired by [cmp-git](https://github.com/petertriho/cmp-git).
