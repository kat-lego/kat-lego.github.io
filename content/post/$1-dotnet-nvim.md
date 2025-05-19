+++
author = "katlego modupi"
title = "dotnet-nvim"
date = "2025-05-18"
description = "nvim plugin for the dotnet command"
tags = [
    "nvim", "dotnet", "lua"
]
+++

A plug in for neovim that interfaces the dotnet cli command.

[![github repo](https://img.shields.io/badge/dotnet_nvim-gray?logo=github)](https://github.com/kat-lego/dotnet-nvim)
<!--more-->

## The Pain of Managing Package References and Solution Files
Neovim is a fantastic editor that offers a keyboard-centric workflow. However, as a .NET developer, managing package references and solution files (.sln) can become cumbersome as I have to switch to a terminal. This post serves to show how neovim's extensibility allows me to simplify the annoyances in my workflow.

## The Issue with Using `:!dotnet <args>` or `:terminal dotnet <args>`
A common workaround to interact with the dotnet CLI from within Neovim is to use the `:!dotnet <args>` or `:terminal dotnet <args>` commands. While functional, these methods have annoying drawbacks. They execute in the context of the directory you launched neovim in, this gets tedious if you have to type out long directory paths.

## Requirements
We want to write a neovim plugin dubbed `dotnet-nvim` that will do the following:
* Execute the dotnet cli command when you run `:Dotnet`.
* The `Dotnet` command should execute with the working directory for the active buffer.
* There should be completions for the subcommands. i.e `dotnet sln`, `dotnet add`, `dotnet restore`,
  etc.
* There should be completions for paths that follow in commands like `dotnet sln add`.

## The stack

### Plugin project structure
First thing is first. The plugin will follow the following structure.

```
dotnet-nvim/
└─── lua/                # Lua modules for the plugin
   └── dotnet-nvim/      # Main Lua file for the plugin
       └─── init.lua     # Initialization and configuration
```

### Installing it on your local nvim lazy vim config
To test our plugin in we will install it using Lazy like so. In this example, I have my config setup
such that each plugin is in its own file.

```lua
return {
  dir = 'path/to/plugin/dotnet-nvim',
  config = function()
    require('dotnet-nvim').setup()
  end,
}
```

### Plugin Creating the Dotnet command
In the `init.lua` file for the plug in, we will add a function called setup, this is a function that 
will be used to setup the plugin. In this function we use the `vim.api.nvim_create_user_command`
function to create a new command and set it up to print 'We are dotnetting'.

```lua
local M = {}

function M.setup()
  vim.api.nvim_create_user_command('Dotnet', function(opts)
    print 'We are dotnetting'
  end, {})
end

return M
```

This should print the text 'We are dotnetting' when you run `:Dotnet` after reloading nvim.

To actually run the `dotnet` command we will add a helper function for running commands to the command
line.
```lua
-- Helper function to run shell commands and capture output
local function run_command(cmd, cwd)
  local handle
  if cwd then
    -- io.popen is used to start a new process to run the command
    handle = assert(io.popen('cd ' .. vim.fn.shellescape(cwd) .. ' && ' .. cmd, 'r'))
  else
    handle = assert(io.popen(cmd, 'r'))
  end

  -- we read the output from the process stdout here
  local result = handle:read '*a'
  handle:close()
  return result
end
```

We will update our setup function to use the `run_command` function as follows:
```lua
function M.setup()
  vim.api.nvim_create_user_command('Dotnet', function(opts)
    local cmd = 'dotnet ' .. table.concat(opts.fargs, ' ')
    local cwd = vim.fn.expand '%:p:h'
    print(run_command(cmd, cwd) .. '\n')
  end, {
    nargs = '*', -- Accept multiple arguments
  })
end
```
  
  1. Notice we have added `nargs = '*'` to the last parameter of `vim.api.nvim_create_user_command`.
  This allows the command to accept multiple arguments
  2. We are using `vim.fn.expand '%:p:h'` to get the path for the current buffer.

And with that we are able to run the dotnet command within nvim.

### Add completions
We will now add a function that will handle the completions. We will now add a function that will
handle the completions. This function takes in user input and provides relevant suggestions based on
the context of the dotnet command being executed.

```lua
local function complete_dotnet(arg_lead, cmd_line)
  local args = vim.split(cmd_line, '%s+')
  local context = args[2] or ''
  local completions = {}

  local cwd = vim.fn.expand '%:p:h'

  if context == 'add' then
    if args[3] == 'reference' then
      completions = get_path_completions(cwd, args[4] or '')
    else
      completions = { 'reference', 'package' }
    end
  elseif context == 'sln' then
    if args[3] == 'add' or args[3] == 'remove' then
      completions = get_path_completions(cwd, args[4] or '')
    else
      completions = { 'add', 'remove' }
    end
  else
    completions = {
      'sln',
      'new',
      'build',
      'run',
      'test',
      'publish',
      'restore',
      'clean',
      'add',
    }
  end

  local matches = {}
  for _, item in ipairs(completions) do
    if item:match('^' .. vim.pesc(arg_lead)) then
      table.insert(matches, item)
    end
  end
  return matches
end
```

Completions for dotnet subcommands are just a static list. We also have completions for file paths
when we run commands like `dotnet sln add`. Here Is The implementation of the `get_path_completions`
function that is used for this.

```lua
-- Helper function to list directories and files under a given path
function get_path_completions(path, prefix)
  local itemsString = vim.fn.glob(path .. '/' .. prefix .. '*')
  local items = vim.split(itemsString, '\n', { trimempty = true })

  local matches = {}

  for _, item in ipairs(items) do
    table.insert(matches, string.sub(item, #path + 2))
  end

  return matches
end

```

We bring all of this together by passing an option in the `vim.api.nvim_create_user_command`
function.
```lua
function M.setup()
  vim.api.nvim_create_user_command('Dotnet', function(opts)
    local cmd = 'dotnet ' .. table.concat(opts.fargs, ' ')
    local cwd = vim.fn.expand '%:p:h'
    local output = run_command(cmd, cwd)
    vim.api.nvim_out_write(output .. '\n')
  end, {
    nargs = '*', -- Accept multiple arguments
    complete = complete_dotnet,
  })
end
```

And this completes the stack. Have a look at the [complete solution](https://github.com/kat-lego/dotnet-nvim) which includes completions for nuget packages and includes some rules on running the command at a solution level for some commands.
