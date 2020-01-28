+++
author = "Anurag Saxena"
categories = ["52p52w", "2018", "ghost"]
date = 2018-02-22
description = ""
draft = false
cover = "https://images.unsplash.com/photo-1509541206217-cde45c41aa6d?ixlib=rb-0.3.5&q=80&fm=jpg&crop=entropy&cs=tinysrgb&w=1080&fit=max&ixid=eyJhcHBfaWQiOjExNzczfQ&s=a73d0ca72c26077a3bc3eb75f31745b9"
slug = "i-switched-to-ghost"
tags = ["52p52w", "2018", "ghost"]
title = "I switched to Ghost"

+++


As you can see, I switched to [ghost](https://ghost.org/). If you go to my [first post](https://anurag.codes/52-posts-in-52-weeks-atleast/), I talked about how I wanted to use posthaven and how it is great.

> I am using posthaven over medium, or blogger, or wordpress, is because of its simplicity. You can read much about posthaven here: https://posthaven.com/ . It has an easy posting format, lets me use my own domain and I can even email my posts. I don’t have to think about it much. Its simple!

While posthaven is simple, it became too simple for me. Posthaven is great if all you have is text and a few images, videos or instagram posts to embed. Posthaven lags when you want to write code snippets in your blogs like so.

```python
import requests
r = requests.get('www.google.com')
print (r.status_code)
```

If you want to write javascript:

```javascript
console.log('Hello World!')
```

I just wrote that using markdown and ghost does a great job of just formatting everything for you if you use markdown. Markdown is great because you do not have to worry about formatting lists, code etc., and is highly flexible. You can check out this [post](https://help.ghost.org/article/4-markdown-guide) on ghost markdown.

The other reason for me to switch over was to have more control on my blog. Since Ghost is open source, I am currently running Ghost on an Ubuntu VM running on Digital Ocean. All of this is running over HTTPS using Lets Encrypt and I am using Cloudflare as my CDN. You can sign up for your own Digital Ocean instance [here](https://m.do.co/c/13dc3b204e08) and get a $10 credit to try out Digital Ocean. After signing up, follow these instructions: https://www.digitalocean.com/community/tutorials/how-to-set-up-the-digitalocean-ghost-one-click-application-for-ubuntu-16-04

You can also use [unsplash for stock imagery on ghost](https://blog.ghost.org/unsplash/), like I did above.

All these features, and control, and more, made me switch to Ghost. I like Posthaven and I like the idea of it. If all I had was a text blog with minimal images, youtube embeds etc, Posthaven is great for that! But I like the level of flexibility and control I have enjoyed so far with ghost.

--

