---
title: "Create a blog using Jekyll and Minimal Mistakes, hosted on Github Pages"
date: 2021-05-21T15:34:30+01:00
permalink: create-a-blog-using-jekyll-with-minimal-mistakes
description: How to create a blog using Jekyll and Minimal Mistakes
categories:
  - blog
tags:
  - Jekyll
---

When searching for "create developer blog" you get a ton of hits. Most of them will recommend a hosting company like Medium or DEV or a personal blog option with hosting like Digital Ocean, Blue etc.

A free hosting option is to use [GitHub pages](https://pages.github.com/). A nice blog site generator that plays well together with GitHub Pages is [Jekyll](https://jekyllrb.com/). Jekyll has loads of themes that lets you customize the look of the blog without having to fiddle with CSS.

Jekyll is not officially supported on Windows so I would recommend going with WSL2 in case you're on a Windows machine.

## Setting things up
Here are the steps for how to create a blog site using Jekyll with a nice theme called [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/) and host it on Github Pages.

* Follow the [Jekyll tutorial](https://jekyllrb.com/docs/step-by-step/01-setup/) in order to get a basic understanding of it.
* Install Jekyll on Ubuntu as described [here](https://jekyllrb.com/docs/installation/ubuntu/)
* Create a public repository on GitHub named \<your-github-username\>.github.io
* Clone the newly created repository to your local machine.
* To start with a nice template, copy the contents of [Minimal Mistakes remote theme starter](https://github.com/mmistakes/mm-github-pages-starter) into your cloned repository.
* Now change the contents and the [configuration](https://mmistakes.github.io/minimal-mistakes/docs/configuration/) of the site to your liking.
* Start a local instance of the site with `bundle exec jekyll serve`

## Using a custom domain name

When you push your code to GitHub it will automatically build and publish your site to https://\<your-github-username\>.github.io. 
If you want use a custom domain like mysuperduperblog.com, follow these steps:
* Register your domain mysuperduperblog.com at a domain registrar.
* Go to your repo \<your-github-username\>.github.io and then to Settings -> Pages -> Custom domain and type your custom domain. Then click Save. 
* Create an A record with your DNS service that points your domain (e.g. mysuperduperblog.com) to the Github Pages' IP addresses:   
185.199.108.153  
185.199.109.153  
185.199.110.153  
185.199.111.153  
* To have a subdomain such as www.mysuperduperblog.com configure a CNAME record at your DNS service. It should point the subdomain to \<your-github-username\>.github.io.

If you've followed these steps you should now have your blog at your chosen domain.