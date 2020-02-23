---
layout: default
title: Build your own text editor in Go
---
# Build your own text editor in Go
Welcome! This is an tutorial that shows you how to build a text
editor in [Go Programming Language](https://www.golang.org).

The text editor is a port of [antirez's kilo](http://antirez.com/news/108)
in Go. Like the C original it's about 1000 lines of Go with no dependencies,
and it implements all the basic features you expect in a minimal editor
including a search feature. This tutorial is adapted from the original
[Snaptoken Tutorial](https://viewsourcecode.org/snaptoken/kilo/) for
the C language version of kilo.

This tutorial walks you through building the editor in **184 steps**. Each step,
you'll add, change, or remove a few lines of code. Most steps, you'll be able
to **observe the changes** you made by compiling and running the program
immediately afterwards. I explain each step along the way, sometimes in 
a lot of detail. Feel free to skim or skip the prose, as the main point 
of this is that **you are going to build a text editor from scratch**! 
Anything you learn along the way is bonus, and there's plenty to learn 
just from typing in the changes to the code and observing the results.

See the [appendices](08.appendices.html) for more information on the 
tutorial itself (including what to do if you get stuck,
and where to get help).

If you're ready to begin, then go to [chapter 1](/setup.html)!