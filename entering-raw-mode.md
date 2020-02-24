# Entering Raw Mode

### Step 1: Initializing a go module for development
Go versions from v1.11 onwards have support for modules to manage
dependencies. For your project choose a module path of your own 
(looking like `github.com/gokilo/gokilo` below) and initialize
a Go module. If you are using version control, just use the part
of your repository URL that looks like above.

**Step 1. Initiate a go module**
```
$ go mod init example.com/user/gokilo
```

### Step 2: (Optional) Add a `.gitignore` file to your project
If you're version controlling using Git, now is  good time to
addd a `.gitignore` file to ensure that generated binaries don't
get checked in by mistake. Even if you're not using version
control just now, you might as well create the `.gitignore` file
in case you do so at a later date.

**Step 2: add a .gitignore file**

**File: .gitignore**
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