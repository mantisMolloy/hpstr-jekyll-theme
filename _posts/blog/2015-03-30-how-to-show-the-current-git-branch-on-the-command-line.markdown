---
layout: post
title: "How to Show the Current Git Branch on the Command Prompt"
modified:
categories: blog
description: getting git branch on the command promt
excerpt: One of the only things I liked about using git bash in windows was that its always showed the current branch at the command prompt. This was lacking in Ubuntu so I decided to try get this functionality at my command prompt. 
tags: [git, hacks, ubuntu ]
image:
  feature: code.jpg
date: 2015-03-30T19:53:57+01:00
share: true
---
One of the things I liked about using gitbash for windows was how it always showed the current branch at the prompt. To make life easier for ourselves we can get the same functionality in ubuntu by simply editing the .bashrc file.

Firstly open up your .bashrc file

{% highlight bash %}
$ vi ~/.bashrc
{% endhighlight %}

Now enter the following variables and function

{% highlight bash %}
YELLOW="\[\033[0;33m\]"
GREEN="\[\033[0;32m\]"
NO_COLOR="\[\033[0m\]"

function parse_git_branch () {
  git branch 2> /dev/null | sed -e '/^[^*]/d' -e 's/* \(.*\)/ (\1)/'
}

PS1="$GREEN\u@\h$NO_COLOR:\w$YELLOW\$(parse_git_branch)$NO_COLOR\$ "
{% endhighlight %}

Don't forget to source your.bashrc file or to log out and log back in for these changes to take effect.

The result is shown in the screengrab below. Now with the current branch always shown there are no excuses for merging into the wrong branch.

![git image]({{site.url}}/images/git1.jpg)
{: .center}

p.s. I do not recommend the "yolo" commit!