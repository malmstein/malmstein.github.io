---
layout: post
title: "Blogging again"
date: 2014-06-07 21:07:34 +0200
comments: true
categories: octopress howto
---

After a couple of years being lazy and just reading what others wrote, I belive it's time to start blogging again. One could explain inspring stories about
blogging and black holes but I'm just going to explain why I chose Octopress over other blogging platforms and how I made it work for me.

<!-- more -->

## Why?

There are several blogging platforms out there, they all have their features and some of them are easier to use. The likes of [Wordpress](https://wordpress.com/), [Ghost](https://ghost.org/), [Medium](https://medium.com/), etc...
are more than good options in order to start. In my case, I used Wordpress in the past, still use it for some other projects but I wanted to use something new for me.

Markdown files are a great way of writing text, it won't be the best option if you are trying to write a novel but if you are familiar with Github you'd probably
have used them in the past. Octopress text editing is based on Markdown files, that was a plus. Octopress also supports Github Pages and the whole publishing process
is based on Git, what else could you ask?

## Installing Octopress


1. Install [Git](http://git-scm.com/)
2. Install Ruby (1.9.3p194 using [rbenv](http://octopress.org/docs/setup/rbenv/) when writing this post)
3. Clone [Octopress](http://octopress.org/)

{% codeblock lang:bash %}
$ git clone git://github.com/imathis/octopress.git octopress
$ cd octopress
{% endcodeblock %}

Next, install dependencies.

{% codeblock lang:bash %}
$ gem install bundler
$ rbenv rehash
$ bundle install
{% endcodeblock %}

Install the default Octopress theme.

{% codeblock lang:bash %}
$ rake install
{% endcodeblock %}

At this point you have Octopress installed and the default theme installed, now we need to deploy it. Octopress supports deployment on GitHub pages
and that's the method I chose.

## Deploying to Github Pages


Github allows you to host a blog from `http://username.github.io`, where `username` is your Github username.

Github Pages for users and organisations uses the master branch like the public directory on a web server, serving up the files at your Pages url http://username.github.io.
As a result, you'll want to work on the source for your blog in the source branch and commit the generated content to the master branch. Octopress has a configuration task that helps you set all this up.

{% codeblock lang:bash %}
$ rake setup_github_pages
{% endcodeblock %}

The rake task will ask you for a URL of the Github repo. Copy the SSH or HTTPS URL from your newly created repository (e.g. git@github.com:username/username.github.io.git) and paste it in as a response.

Done, at this point your Github page is ready to be deployed, but first we need to generate the blog.

{% codeblock lang:bash %}
$ rake generate
$ rake deploy
{% endcodeblock %}

This will generate your blog, copy the generated files into _deploy/, add them to git, commit and push them up to the master branch. In a few seconds you should get an email from Github telling you that your commit has been received and will be published on your site.

As you would do with any other git based project, after adding some code you'd add it and commit it.

{% codeblock lang:bash %}
$ git add .
$ git commit -m 'your commit message'
$ git push origin source
{% endcodeblock %}

## Using a custom domain


First thing that needs to be added is a `CNAME` in the blog's source.

{% codeblock lang:bash %}
$ echo 'www.malmstein.com' >> source/CNAME
{% endcodeblock %}

I'm using a main domain, so all I had to do is modifying the DNS host and add a new `CNAME` record for `www` pointing to `malmstein.github.io`

Done!

Further reading and more detailed instruction can be found in the [Octopress](http://octopress.org/docs/configuring/) documentation.
There is also a very useful [cheat sheet](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet) for markdown files.
