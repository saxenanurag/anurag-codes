+++
author = "Anurag Saxena"
categories = ["52p52w", "2018", "git", "github", "code"]
date = 2018-07-24
description = ""
draft = false
cover = "https://images.unsplash.com/photo-1468487422149-5edc5034604f?ixlib=rb-0.3.5&q=80&fm=jpg&crop=entropy&cs=tinysrgb&w=1080&fit=max&ixid=eyJhcHBfaWQiOjExNzczfQ&s=53c38d18e3bded29e2ae1079b7db7355"
slug = "git-stash"
tags = ["52p52w", "2018", "git", "github", "code"]
title = "Git stash"

+++


Git stash allows you to save your changes to a repo, without any commits, and go back to the clean state that your repo was in since the last commit. I use stash when I have to pull changes from origin into my code, but I don't want to commit any of my own changes. Stash allows to manage merge conflicts as well. You can use stash when you want to switch branches. 

Note that when you stash, your changes are "stashed" with a reference id on it. 

To stash:
```
git stash
```
To unstash:
```
git stash apply
```
When you `apply` the stash, it does not remove the stash, or the state, from the stash list. 
If you want to see the stash list, you can: 
```
git stash list
```
If you want to apply the stash to the current working branch and remove the stash from the list, you can: 
```
git stash pop
```

Here is a scenario. You are working on changes, you realize that you are working on the master branch when you should have been working on the feature branch. But you dont want to get rid of all the work you have done to switch branches. What do you do?
Solution: 
```
git stash
git checkout <feature-branch>
git stash pop
```
Profit!

