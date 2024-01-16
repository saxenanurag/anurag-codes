+++
author = "Anurag Saxena"
categories = ["52p52w", "til", "vim", "2018", "code"]
date = 2018-11-09
description = ""
draft = false
cover = "https://images.unsplash.com/photo-1488903809927-48c9b4e43700?ixlib=rb-0.3.5&q=80&fm=jpg&crop=entropy&cs=tinysrgb&w=1080&fit=max&ixid=eyJhcHBfaWQiOjExNzczfQ&s=cb668159df68b470041dbf223fdf415f"
slug = "til-creating-a-new-file-or-folder-in-nerdtree"
tags = ["52p52w", "til", "vim", "2018", "code"]
title = "TIL: Creating a new file or folder in NerdTree"

+++


To create a new file or folder in NerdTree, bring up the NerdTree menu by typing `m`. This brings up a multiple options including:

```
(a)dd a childnode
(m)ove the current node
(d)elete the current node
(r)eveal in Finder the current node
(o)pen the current node with system editor
(q)uicklook the current node
(c)opy the current node
(l)ist the current node
```

So now you press `a` and can type in the file name or the folder that you want to create. 

IF you decide you don't want to create or use any of the menu options and get out of this menu, a simple `CTRL+C` will get you out. 

Always remember: `q!` to quit vim - if you have one file open. `qa!` to close all files and quit!



