---
title: 自用TMUX配置
id: '296'
tags: []
categories:
  - - uncategorized
comments: false
date: 2022-05-04 17:55:42
---

```
set -g prefix2 C-a
bind C-a send-prefix -2

set -g escape-time 0
set-option -g mouse on

setw -g mode-keys vi
bind Escape copy-mode
bind -T copy-mode-vi v send-keys -X begin-selection
bind -T copy-mode-vi y send-keys -X copy-selection-and-cancel
bind p pasteb

unbind '"'
unbind '%'
bind c new-window -c "#{pane_current_path}"
bind C-c kill-window
bind \\ splitw -h -c "#{pane_current_path}"
bind - splitw -v -c "#{pane_current_path}"
bind C-n kill-pane

# move around
bind -r h select-pane -L
bind -r l select-pane -R
bind -r j select-pane -D
bind -r k select-pane -U
bind -r C-k resizep -U 10
bind -r C-j resizep -D 10
bind -r C-h resizep -L 10
bind -r C-l resizep -R 10

# switch windows
bind -r C-e last

```