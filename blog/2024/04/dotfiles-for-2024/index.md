---
layout: blog
title: Dotfiles for 2024
date: 2024-04-16T11:56:51-0500
lastMod: 2024-04-16T11:58:23-0500
categories: workflow
tags:
  - dotfiles
  - vim
  - tmux
  - alacritty
  - neovim
  - github
description: Going over my dotfiles setup for 2024. Here's to optimizing my workflow!
disableComments: true
draft: true
---

## Intro

I've been on this kick to update my [dotfiles](https://github.com/sinicide/.dotfiles) so that I can also revamp my dotfiles that I use for work as well in an effort to optimize my workflow. With this I'm also making the switch from Vim to Neovim as my default text editor in order to really force myself to use and learn vim motions more. I'm not a software engineer, so my typical usage with Vim has just been to edit configuration files, write bash scripts or perform small edits. I've been using VSCode a lot for writing configuration yamls and anything remotely close to programming. So I'm going cold turkey and cutting VSCode out of my repertoire.

So let's go over some goals I have for iterating over my existing dotfiles.

1. Creating a new branch for the update before merging to master.
2. Adding new configurations for neovim and tmux.

But before we go all in on what I'm adding, let's go over a bit on how my dotfiles are structured for modularity.

## The basic structure

It was a few years ago that I learned about GNU [stow](https://www.gnu.org/software/stow/manual/stow.html) which allows you to create and manage symlinks. With this we can manage symlinks appearing in a given location following the structured format of another.

To illustrate this, let's say we have the following directory structure in my home directory

```
drwx--x---+ 53 sinicide sinicide 4.0K Apr 14 18:37 .
drwxr-xr-x   3 root     root     4.0K Sep 16  2021 ..
drwxr-xr-x   2 sinicide sinicide 4.0K Feb 17 20:41 bin
drwxr-xr-x  72 sinicide sinicide 4.0K Apr 14 15:55 .config
drwxr-xr-x  10 sinicide sinicide 4.0K Apr  5 07:41 .dotfiles
drwxr-xr--+ 30 sinicide sinicide  80K Apr 14 13:37 Downloads
lrwxrwxrwx   1 sinicide sinicide   29 Jan 23  2022 .gitconfig -> .dotfiles/personal/.gitconfig
drwxr-xr-x   3 sinicide sinicide 4.0K Aug 11  2023 Pictures
drwxr-xr-x   2 sinicide sinicide 4.0K Dec  7 16:52 .ssh
lrwxrwxrwx   1 sinicide sinicide   20 Mar 30 00:28 .tmux -> .dotfiles/tmux/.tmux
lrwxrwxrwx   1 sinicide sinicide   25 Mar 30 00:28 .tmux.conf -> .dotfiles/tmux/.tmux.conf
lrwxrwxrwx   1 sinicide sinicide   18 Mar 29 14:42 .vim -> .dotfiles/vim/.vim
-rw-------   1 sinicide sinicide  32K Mar 29 15:39 .viminfo
lrwxrwxrwx   1 sinicide sinicide   20 Mar 29 14:42 .vimrc -> .dotfiles/vim/.vimrc
-rw-------   1 sinicide sinicide  21K Oct 17  2021 .zhistory
-rw-r--r--   1 sinicide sinicide   87 Mar 13 00:26 .zlogin
lrwxrwxrwx   1 sinicide sinicide   21 Mar 29 14:42 .zshenv -> .dotfiles/zsh/.zshenv
-rw-------   1 sinicide sinicide 349K Apr 14 18:37 .zsh_history
lrwxrwxrwx   1 sinicide sinicide   20 Mar 29 14:42 .zshrc -> .dotfiles/zsh/.zshrc
```

From this you can see how the symlinks are mapped to the files within my `.dotfiles` directory, I've split up configurations into their own specific directories, such as `tmux`, `zsh`, and `vim` So the base of these subdirectories act as the base home directory and how I structure the files and subdirectories within this maps back to the home directory.

```
/home/sinicide/ = /home/sinicide/.dotfiles/zsh/
```

This makes it very modular and if I want to add new configs that are specific to an application I can just create a new directory for it. So for example, I'll be adding Neovim configs which will live in

```
/home/sinicide/.dotfiles/nvim/
```

Neovim configs are stored within a user's `.config` directory. So we'll structure them the same way within our sub directory and when we run the stow command it'll map it accordingly in our home directory.

```
/home/sinicide/.dotfiles/nvim/.config/nvim/
```

How `stow` works is that it can take a target directory, which is where `stow` should create the symlinks and then it takes a given directory that it maps.

```
cd ~/.dotfiles
stow -t ~ nvim
```

So the above command here will take our Home directory as a target and will map our `nvim` directory. So when I navigate into the `~/.config` directory I'll see the following symlink.

```
nvim -> ../.dotfiles/nvim/.config/nvim
```

## Neovim

So one of my biggest undertaking here is switching to Neovim, with this comes a slew of possible plugins and key remaps. I had the following goals in mind to help make my choices.

1. Keep key remaps to a minimum to start
2. Choose plugins that are useful instead of just providing a visual cool factor

With that said, let's take a look at my nvim config directory and how I've structure it.

```
[sinicide        4096]  .
├── [sinicide          42]  init.lua
├── [sinicide        2123]  lazy-lock.json
├── [sinicide        4096]  lua
│   ├── [sinicide        4096]  core
│   │   ├── [sinicide          42]  init.lua
│   │   ├── [sinicide         591]  remap.lua
│   │   └── [sinicide         915]  set.lua
│   ├── [sinicide        4096]  packagemanager
│   │   └── [sinicide         494]  init.lua
│   └── [sinicide        4096]  plugins
│       ├── [sinicide         902]  conform.lua
│       ├── [sinicide        4591]  lsp-config.lua
│       ├── [sinicide         211]  markdown-preview.lua
│       ├── [sinicide         955]  mason.lua
│       ├── [sinicide         281]  monokai-pro.lua
│       ├── [sinicide        1252]  nvim-cmp.lua
│       ├── [sinicide         954]  nvim-lint.lua
│       ├── [sinicide         891]  telescope.lua
│       ├── [sinicide         906]  treesitter.lua
│       ├── [sinicide          91]  undotree.lua
│       ├── [sinicide         111]  vim-paper.lua
│       └── [sinicide         528]  vim-tmux-navigator.lua
└── [sinicide        4096]  undodir
```

At the core of it, we have the `init.lua` inside nvim, which starts off all the requirements. This is a pretty simple file that only contains 2 things for my setup

```
require("core")
require("packagemanager")
```

The required just points to non-plugin configurations, which are essentially the vim configurations and remaps. The other one just points to [lazy.nvim](https://github.com/folke/lazy.nvim) plugin manager. Now I do want to note that the docs recommends structuring the lazy config within the top level `init.lua`, `plugins.lua` or `plugins/init.lua` However I structured mine within a directory I called `packagemanager` and called it from the `init.lua` just to help me understand the structure of loading things in neovim.

### nvim core configs

So the `core` directory here contains 2 main things, vim default configs and key remappings, the `init.lua` just points to 2 these two files. I won't go into detail on the vim configs, these are primarily carried over from my original vim dotfiles and are less interesting I think.

As for the key remaps, I do want to touch base on them since I'm trying to use VIM motions more and more so I've kept my remaps to a minimum for now to ensure that I learn the remaps as muscle memory.

I've taken these remaps from [ThePrimeagen](https://github.com/ThePrimeagen/init.lua/blob/master/lua/theprimeagen/remap.lua) I did a little testing here and there to determine their benefits for my workflow. So let's go over the ones I've chosen.

```
-- page up/down, keeping cursor centered
vim.keymap.set("n", "<C-d>", "<C-d>zz")
vim.keymap.set("n", "<C-u>", "<C-u>zz")
```

I like the idea of keeping focus towards the center of the screen, so basically this remaps combines the default page up/down and then centers the cursor in the middle of the screen.

```
-- search, but keep item middle of screen
vim.keymap.set("n", "n", "nzzzv")
vim.keymap.set("n", "N", "Nzzzv")
```

The following keeps this same thing in mind, which is to keep the search pattern in center focus.

```
-- yank/paste to/from system clipboard
vim.keymap.set({"n", "v"}, "<leader>y", [["+y]])
vim.keymap.set("n", "<leader>p", [["+p]])
-- delete into blackhole register
vim.keymap.set({"n", "v"}, "<leader>d", [["_d]])
```

The above here should keep 2 separate copy buffers, one for vim and one for system clipboard.

```
-- shift highlighted line up/down
vim.keymap.set("v", "J", ":m '>+1<CR>gv=gv")
vim.keymap.set("v", "K", ":m '<-2<CR>gv=gv")
```

Finally the above remap seems super nice for being able to highlight a block of text and shifting it up or down without having to yank/paste.

### The plugins

Let's talk plugins, I'm keeping my approach with this to the absolute minimal for now, so you can see that I'm not installing a filemanager like NERDTree and just sticking with the default `netrw` while it ain't pretty, it does get the job done. I'm also not doing much in the way of customizing the bottom bar either, no lualine or anything as it's really just for aesthetics.

I'm using [telescope](https://github.com/nvim-telescope/telescope.nvim) to fuzzy find my way around files, [undotree](https://github.com/mbbill/undotree) for viewing previous iterations of a file, this is particularly helpful for configuration orientated files which I find myself working with often in my workflow. [vim-tmux-navigator](https://github.com/christoomey/vim-tmux-navigator) is for some nice keybindings for navigating between vim and tmux. [Markdown Preview](https://github.com/iamcco/markdown-preview.nvim) for being able to view what my markdown looks like rendered in a web browser, this is useful in particular for this blog as all content is written in Markdown, previously I've been using something similar in VSCode, so this is totally a nice to have.

Finally we have 2 themes, monokai-pro is my default theme, but I have a nice vim-paper one for light mode if I ever need to do screensharing/presenting that way vim in my terminal is actually readable.

The rest are for LSPs, linting and formatting, with the use of [treesitter](https://github.com/nvim-treesitter/nvim-treesitter), [mason](https://github.com/williamboman/mason.nvim) for managing LSPs, and [conform](https://github.com/stevearc/conform.nvim) for managing Linters/Formatters.
Mason is pretty awesome for being able to manage installing LSPs, Linters, and Formatters, however the only thing it can auto install it seems are LSPs. Hence why we have Conform which helps to automating the installation of Linters and Formatters.

With that my Neovim setup is complete.

## Tmux Configs

My tmux configs are split between 2 main components, the `.tmux.conf` configuration file and a custom theme.

Let's start by talking about the configuration file as this primarily contains my key bindings. Like most people, I've rebind the leader prefix from the default `C-b` to `C-a`

Most important rebinds I have are, changing the pane navigation to VIM movements ala `hjkl` and the splits. I've opted to use `|` for horiztonal splits and `-` for vertical splits, simply because they look like their respective cuts as opposed to the defaults, which are `%` and `"` , while it wouldn't be difficult to learn them over time, my visual splits are just easier for me to remember.

```
# rebind navigation to VIM keys
bind -r h select-pane -L # move left
bind -r j select-pane -D # move down
bind -r k select-pane -U # move up
bind -r l select-pane -R # move right

# rebind splits
unbind |
unbind -
unbind %
unbind '"'
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"
```

Like most, I also change the default numbering from starting 0 to 1. The reasoning behind this is that the 1 key is just closer on the left hand, so it's a bit more optimized for session switching, as opposed to using 2 hands to hit the prefix keys + session number.

I also have the configurations in place for loading the Tmux Plugin Manager, however I'm not currently using any plugins at the moment. I was testing out the vim-tmux-navigator, but have decided against the keybinds that it default for moving windows, which is to not use leader key prefix and only relay on `<ctrl> + <key>`. Personally I like keeping this leader key prefix style specific for tmux so that I have it separate in my mind. With the keybindings it also removes the clear line key mapping in the terminal, so that was the main motivator for me to remove this plugin. I don't need to necessarily use tmux all the time, so I don't need to keep 2 different key bindings in my brain for the same action.

Finally we have my custom theme, I looked at a bunch of the other themes out there for tmux and honestly I didn't care for what they offered. So of course I had to create my own. So what makes my theme different? Mostly the removal of hostname + time as these are unnecessary give modern desktops generally show a clock somewhere.

So I thought about the type of information that would be important to me with managing multiple terminal windows. This essentially boils down to,

1. What Session am I on?
2. What Window am I on?
3. How many panes are in a given window?
4. Am I zoomed into a pane?

So I've customized the theme with these in mind and have removed the necessary things.

```
# left status bar
set-option -g status-left-style 'bg=#19181a'
set-option -g status-left ' '

# right status bar
set-option -g status-right-length 40
set-option -g status-right-style 'bg=#19181a,fg=#a9dc76'
set-option -g status-right '#{?client_prefix,#[bg=#fc9868]#[fg=#19184a],} [#{=30:session_name}] '

# windows status
set-option -g window-status-format '#{window_index}#(echo ":")#{=20:#{b:window_name}}#{window_flags}'
# window current/active status
set-option -g window-status-current-format '#{?client_prefix,#[bg=#fc9868]#[fg=#19184a],}#{window_index}:#(echo "#{b:window_name}")#{?window_zoomed_flag,[Z],[#{window_panes}]}#{?mouse,[M],}'
```

So you can see that this is split into 3 sections, a left side, right side and window-status, gets pushed to the left side. A nice little addition I added was the `Z` indicator whenever I'm zoomed into a pane.

## Conclusion

With these new changes implemented, I can `git pull` and update my existing dotfiles and stow the new changes all at once or one at a time. This becomes quite flexible when I want to integrate a personal or work folder given whether I'm using my work computer or my personal computer.

All and all dotfiles are great and no two need be alike. I quite enjoy seeing how people setup their own dotfiles and take inspiration and snippets where it makes sense for my own workflow.
