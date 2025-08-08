---
layout: post
title: The Tmux Tool
date: 2025-08-02 18:11:00
description: Teaching myself to use the terminal multiplexer
tags: code developer
categories: tech
---

# Terminal Multiplexer

The first time I encountered `tmux` was in my systems CS class. My professor was in the middle of an in class demonstration and mentioned that he liked to use this tool in the terminal called `tmux` because it made work a lot easier. He then proceeded to call `tmux` to manage his terminal session and run a bunch of code side by side. The whole thing was effortless and just looked like magic to me. I remember thinking to myself, "wow that IS really useful, maybe I should go learn how to use it myself". Unfortunately I never quite found the time to - it was one of those things where once you put it to the side, you end up never coming back to it. I had forgotten about it until recently when I began to feel frustrated with my own workflow and the hassle of constantly creating new terminal windows and just going in and out of `vim` and other tools in the command line.

{% include figure.liquid loading="eager" path="assets/img/tmuxcover.png" class="img-fluid rounded z-depth-1" zoomable=true %}

# Commands

To start a `tmux` server with the default session (session 0), just run:

```shell
tmux
```

To detach from your current `tmux` session the key stroke is **Ctrl+B** then **D** (it's a little awkward so people do configure this). The **Ctrl+B** key stroke patterns only work inside active `tmux` sessions. To see all `tmux` sessions use:

```shell
tmux ls
```

The cool thing about `tmux` is that when sshing onto a server, the tmux process can be started on the server. With the background process running, even if you disconnect from ssh and then come back and reattach, the process will still be running! You can simply reconnect to the server and reattach to the session with:

```shell
tmux attach -t 0
```

You can abbreviate:

```shell
tmux a -t 0
```

To create a new session and name it:

```shell
tmux new -s mysession
```

To kill a session:

```shell
tmux kill-sess -t mysession
```

## Sessions, Windows, and Panes

It's useful to have the model of `session -> window -> pane`.

You can have multiple windows in each session (think of windows as tabs in a browser). You can add new windows with **Ctrl+B** followed by **C**. Jump to the specified window number with **Ctrl+B** followed by **2** for window 2 or use **Ctrl+B** followed by **N** or **P** to go forward or back. To kill the window use **Ctrl+B** then **&**.

Within each of these windows you can also split the session window into horizontal or vertical panes using **Ctrl+B** followed by **%** for horizontal and **"** for vertical. **Ctrl+B** with arrow keys then moves between the panes. Users often recommend enabling mouse support for `tmux` by modifying the config file so you can resize panes more easily.

One of the shortcuts to be careful of is **Ctrl+D** which exits (terminates) that `tmux` session you are in. It's easy to accidentally hit that key stroke while trying to detach.

{% include figure.liquid loading="eager" path="assets/img/tmux.png" class="img-fluid rounded z-depth-1" zoomable=true %}

<div class="caption">
    Cheatsheet of all tmux commands.
</div>

Another feature to be aware of is the ability to select text in your windows and then copy and paste them. The `extracto` plugin is nice because it allows you to fuzzy select and not have to manually select it.

# Plugins & Themes

There are also some cool themes and useful plugins out there for configuring the `tmux` experience:

- [extracto](https://github.com/laktak/extrakto)
- [arctic theme](https://github.com/nordtheme/tmux)

I would also recommend using the `tmux` plugin manager [here](https://github.com/tmux-plugins/tpm) instead of manually downloading them.

My `.tmux.conf` file looks like this:

```
# split panes using | and -
bind | split-window -h
bind - split-window -v
unbind '"'
unbind %

# switch panes using Alt-arrow without prefix
bind -n M-Left select-pane -L
bind -n M-Right select-pane -R
bind -n M-Up select-pane -U
bind -n M-Down select-pane -D

# enable mouse
set -g mouse on

# List of plugins
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible'
set -g @plugin 'laktak/extrakto'
set -g @plugin 'wfxr/tmux-power'

# Initialize TMUX plugin manager (keep this line at the very bottom of tmux.conf)
run '~/.tmux/plugins/tpm/tpm'
```

Note: setting up extrakto requires `fzf` (this involves downloading with brew, adding a line to `.bashrc`).

# Resources

Some tutorials I consulted:

- [Red Hat guide](https://www.redhat.com/en/blog/introduction-tmux-linux)
- [Dev tutorial](https://dev.to/iggredible/tmux-tutorial-for-beginners-5c52)
- [Customizing Tmux](https://hamvocke.com/blog/a-guide-to-customizing-your-tmux-conf/)
