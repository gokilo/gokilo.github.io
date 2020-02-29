---
layout: default
title: Setup
---

Ahh...  Don't you love a fresh start on a blank slate?

Unfortunately, when you're building a *computer program*, progressing
from the blank slate can get... complicated. And frustrating. You have
to make sure your environment is set up for the programming language 
you're using, and you have to figure out how to compile and run your 
program in that environment.

Fortunately, the program we are building doesn't depend on any external
libraries, so you don't need anything beyond the Go compiler and the standard
library and some tooling.

# Pre-Requisites

## Some flavor of Unix or Windows Sub-system for Linux
This tutorial interacts a lot with the terminal and assumes you will be
working in a Unixy environment. If you are on a Windows 10 machine, it is
recommended you install [Windows Sub-system for Linux](https://docs.microsoft.com/en-us/windows/wsl/about) which gives you a Linux environment without the
overhead of virtual machines.

GoKilo does compile and work perfectly in a native windows console like `cmd`
or `powershell` but the required code modifications will be covered in a 
separate appendix in the interest of pedagogical simplicity.

A few tips if you use Windows Subsystem for Linux
- WSL seems to open in the `system32` directory by default. To open
  in the home directory
  Simply create a shortcut that has the command line something like 
  `C:\Windows\System32\wsl.exe ~` to open in the home directory.
- Typing `explorer .` on the WSL command line opens an explorer window
  that mounts the WSL home directory.
- Typing `code .` opens Visual Studio Code and loads the WSL current
  directory in the workspace if you have it installed

## Setup the Go programming language distribution
[Install the Go language distribution](https://golang.org/doc/install) 
on your development machine. Go 1.13 or higher is preferred. In many
Linux distributions, the default Go version may be dated and so it 
is recommended to install from the official website.

## Setup your development environmnet for Go
Most modern text editors and many IDEs offer enhanced development 
support for Go like syntax highlighting, code auto-formatting on save
and auto-completion. Choose the programming text editor or IDE you
are be most familiar with and install Go language support:
- Official [Editor Plugins and IDEs](https://golang.org/doc/editors.html) page
- [Community maintained Editors and IDEs wiki](https://github.com/golang/go/wiki/IDEsAndTextEditorPlugins)

## Get familiar code the development processs in Go
This tutorial assumes you have a working knowledge of Go and will not
teach basic language concepts and development workflows. If you are
new to Go, please ensure you have gone through at least the following:
- [A Tour of Go](https://tour.golang.org)
- [How to write Go Code](https://golang.org/doc/code.html)

You may also find [Go by Example](https://gobyexample.com/) and 
[Effective Go](https://golang.org/doc/effective_go.html) to be valuable
learning resources.

# Getting ready for Development

## 0. Setup your development directory
It is recommended that you create a remote git repository
(eg. [GitHub](https://github.com/), [GitLab](https://about.gitlab.com/),
[BitBucket](https://bitbucket.org/) etc. all have free tiers) to
version control your source code, recover from hardware failures
and revert in case of major errors. If you have already done so,
then start your development by cloning the repository.
```
$ git clone <path to your repository>
```
Alternatively, create a directory for your project.
```
$ mkdir gokilo
```

## 1. Initializing a go module for development
Go versions from v1.11 onwards have support for modules to manage
dependencies. For your project choose a module path of your own 
(looking like `github.com/gokilo/gokilo` below) and initialize
a Go module. If you are using version control, just use the part
of your repository URL that looks like above.


| **Commit Title** | **File** |
|:-----------------|---------:|
| 1. Initiate a go module | |

```
$ go mod init github.com/gokilo/gokilo #replace repo path with your own
```

**Note**: From this step onwards, you can follow the code in the
[`https://github.com/gokilo/gokilo`](https://github.com/gokilo/gokilo)
repo. The commit titles in that repo will correspond to the ones shown
in the table above each step.

## 2. (Optional) Add a `.gitignore` file to your project
If you're version controlling using Git, now is  good time to
addd a `.gitignore` file to ensure that generated binaries don't
get checked in by mistake. Even if you're not using version
control just now, you might as well create the `.gitignore` file
in case you do so at a later date.

| **Commit Title** | **File** |
|:-----------------|---------:|
| 2. add a .gitignore file | .gitignore |

```
# Binaries for programs and plugins
*.exe
*.exe~
*.dll
*.so
*.dylib

# Test binary, built with `go test -c`
*.test

# Output of the go coverage tool
*.out

# Vendored dependencies
# vendor/

gokilo
```


We're ready to begin! Head on to [Entering Raw Mode](/entering-raw-mode.html)