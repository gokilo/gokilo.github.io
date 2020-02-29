---
layout: default
title: Entering Raw Mode
---

# Entering Raw Mode

Let's try and read keypresses from the user. 

| **Commit Title** | **File** |
|:-----------------|---------:|
| 3. Get keypresses from user| main.go|

```go

package main

import (
	"bufio"
	"fmt"
	"io"
	"os"
)

func main() {

	r := bufio.NewReader(os.Stdin)

	for {
		_, err := r.ReadByte()

		if err == io.EOF {
			break
		} else if err != nil {
			fmt.Printf("Error reading from Stdin: %s\n", err)
			os.Exit(1)
		}
	}
}

```

In Go, the standard input device is exposed in the `os` package as 
package level variable called `os.Stdin` which is of the `os.File`
type. Rather than directly reading from the file C-style, we will prefer to
use a buffered i/o reader provided by Go's standard `bufio` package 
which automatically buffers key presses and provides convenient
functions to read or peek a byte at a time. Since `os.File` 
implements the standard `io.Reader` interface, creating a buffered
i/o reader is as simple as calling `bufio.NewReader(os.Stdin)`
to wrap it up in a buffered i/o reader.

When you compile and run `./gokilo`, your terminal gets hooked up to
the standard input, and so your keyboard input gets read in the call
to `r.ReadByte()`. Since at this stage, we are not actually using the
character we read, we assign it to an `_` which throws away the 
return value as Go compiler doesn't allow un-used variables. 

In Go, errors are returned as values rather than as thrown exceptions 
etc. So here we check for errors before continuing the infinite loop
of reading from the keyboard. Specifically, we check if the `err` variable
is set to `io.EOF` which marks the `end of file` marker and if so, we 
exit the program normally (remember Stdin is a `os.File`, you could 
theoritically pipe from some other device to it). If the error is not 
nil but some other value, we exit with an error message & code.

By default your terminal starts in **canonical mode**, 
also called **cooked mode**. In this mode, keyboard input is only sent to
your program when the user presses <kbd>Enter</kbd>. This is useful for
many programs: it lets the user type in a line of text, use <kbd>Backspace</kbd>
to fix errors until they get their input exactly the way they want it, and
finally press <kbd>Enter</kbd> to send it to the program. But it does not
work well for programs with more complex user interfaces, like text editors.
We want to process each keypress as it comes in, so we can respond to it
immediately.

What we want is **raw mode**. Unfortunately, there is no simple switch you can
flip to set the terminal to raw mode. Raw mode is achieved by turning off a
great many flags in the terminal, which we will do gradually over the course of
this chapter. This also works differently for Windows and Linux - so we will
also discuss how Go handles conditional compilation out of the box allowing
you to create a Windows version & a Linux version of the same function.

To exit the above program, press <kbd>Ctrl-D</kbd> to tell the program that it
has reached the end of file. Or you can always press <kbd>Ctrl-C</kbd> to signal
the process to terminate immediately.

## Press <kbd>q</kbd> to quit?

To demonstrate how canonical mode works, we'll have the program exit when it
reads a <kbd>q</kbd> keypress from the user. 

| **Commit Title** | **File** |
|:-----------------|---------:|
| 4. Press q to quit| main.go|

```go

			os.Exit(1)
		}

		//######## Lines to Add ##########

		if b == 'q'{
			break
		}


		//################################
	}
}
```

To quit this program, you will have to type a line of text that includes a `q`
in it, and then press enter. The program will quickly read the line of text one
character at a time until it reads the `q`, at which point the `for` loop
will stop and the program will exit. Any characters after the `q` will be left
unread on the input queue, and you may see that input being fed into your shell
after your program exits.

## Turn off echoing



In Go, interactions with operating systerm primitives are handled by 
operating system specific packages under the [`golang.org/x/sys`](https://godoc.org/golang.org/x/sys) 
namespace . To get and control terminal attributes in Unix, we can
use the `IoctlGetTermios()` and `IoctlSetTermios()` functions from 
the `unix` package which mirror the more traditional `tcgetattr()` 
and `tcsetattr()` functions in the Unix C library.

We can set a terminal's attributes by
1. Reading the current terminal attributes into a `unix.Termios` struct
2. Modifying the value of one or more fields in the struct with bit manipulation
3. Writing back out the modified terminal attributes.

Let's try turning off the `ECHO` feature this way. We will create the 
function to carry out this operation in a new file in the same `main`
package called `rawmode_unix.go` to take advantage of a powerful feature
of go called build tags. In code below, take a look at the very first line
`// +build linux` which tells the Go compiler to only use this file when
building for Linux targets. This way, we can later port GoKilo to Windows
by creating a platform appropriate implementation with `// +build windows` tag 

**Note**: you may use `darwin` or `freebsd` etc. instead of `linux` if you're
following the tutorial on those systems. I haven't tested it personally,
but it should work.

| **Commit Title** | **File** |
|:-----------------|---------:|
| 5. Turn off echoing | rawmode_unix.go|

```go

// +build linux

package main 

import (
	"fmt"

	"golang.org/x/sys/unix"
)

func rawMode() error {

	termios, err := unix.IoctlGetTermios(unix.Stdin, unix.TCGETS)
	if err != nil {
		return fmt.Errorf("could not fetch console settings: %s", err)
	}

	termios.Lflag = termios.Lflag &^ unix.ECHO 

	if err := unix.IoctlSetTermios(unix.Stdin, unix.TCSETSF, termios); err != nil {
		return fmt.Errorf("could not set console settings: %s", err)
	}

	return  nil
}

```


| **Commit Title** | **File** |
|:-----------------|---------:|
| 5. Turn off echoing | main.go|

```go

func main() {

	//######## Lines to Add ##########

	if err := rawMode(); err != nil{
		fmt.Printf("Error: %s\n", err)
		os.Exit(1)
	}

	//################################

	r := bufio.NewReader(os.Stdin)
```

The `unix.ECHO` feature causes each key you type to be printed to the terminal, so
you can see what you're typing. This is useful in canonical mode, but really
gets in the way when we are trying to carefully render a user interface in raw
mode. So we turn it off. This program does the same thing as the one in the
previous step, it just doesn't print what you are typing. You may be familiar
with this mode if you've ever had to type a password at the terminal, when
using `sudo` for example.

Terminal attributes can be read into a `Termios` struct by `IoctlGetTermios()`.
After modifying them, you can then apply them to the terminal using 
`IoctlSetTermios()`. The `unix.TCSETFSF` (ie. Flush) argument specifies
when to apply the change: in this case, it waits for all pending output to
be written to the terminal, and also discards any input that hasn't been read.

The `Termios.Lflag` field is for "miscellaneous flags". The other flag
fields are `Termios.Iflag` (input flags), `Termios.Oflag` (output flags), and
`Termios.Cflag` (control flags), most of which we will have to modify to
enable raw mode.

`unix.ECHO` is a [bitflag](https://en.wikipedia.org/wiki/Bit_field), defined as
`00000000000000000000000000001000` in binary. We use the bitwise-NOT operator
(`~`) on this value to get `11111111111111111111111111110111`. We then
bitwise-AND this value with the flags field, which forces the fourth bit in the
flags field to become `0`, and causes every other bit to retain its current
value. Flipping bits like this is common in systems proramming.

We finally add a call to this function in our `main()` function at the beginning.

After the program quits, depending on your shell, you may find your terminal is
still not echoing what you type. Don't worry, it will still listen to what you
type. Just press <kbd>Ctrl-C</kbd> to start a fresh line of input to your
shell, and type in `reset` and press <kbd>Enter</kbd>. This resets your
terminal back to normal in most cases. Failing that, you can always restart
your terminal emulator. We'll fix this whole problem in the next step.


## Disable raw mode at exit

Let's be nice to the user and restore their terminal's original attributes when
our program exits. We'll save a copy of the `Termios` struct in its original
state, and use `unix.IoctlSetTermios()` to apply it to the terminal when 
the program exits with a new function called `restore()`.

Keeping platform portability in mind, `Termios` is not really defined
on the Windows platform, so rather than have the `rawMode()` function
return it directly, we will serialize it using Go's native `encoding/gob`
package into a byte slice that can be saved by he caller and passed back
to be decoded and restored. Since Go supports multiple return values
from functions, we can return both the byte slice and any error encountered.


| **Commit Title** | **File** |
|:-----------------|---------:|
| 6. Disable raw mode at exit| rawmode_unix.go|

```go

import (
    //######## Lines to Add/Change ##########
    "bytes"
    "encoding/gob"
    //################################

    "fmt"

    "golang.org/x/sys/unix"
)

//######## Lines to Add/Change ##########

func rawMode() ([]byte, error) {

	termios, err := unix.IoctlGetTermios(unix.Stdin, unix.TCGETS)
	if err != nil {
		return nil, fmt.Errorf("could not fetch console settings: %s", err)
	}

	buf := bytes.Buffer{}
	if err := gob.NewEncoder(&buf).Encode(termios); err != nil {
		return nil, fmt.Errorf("could not serialize console settings: %w", err)
	}

	termios.Lflag = termios.Lflag &^ unix.ECHO

	if err := unix.IoctlSetTermios(unix.Stdin, unix.TCSETSF, termios); err != nil {
		return nil, fmt.Errorf("could not set console settings: %s", err)
	}

	return buf.Bytes(), nil
}

// Restore restoes the console to a previous row setting
func restore(original []byte) error {

	var termios unix.Termios

	if err := gob.NewDecoder(bytes.NewReader(original)).Decode(&termios); err != nil {
		return fmt.Errorf("could not decode original console settings: %w", err)
	}

	if err := unix.IoctlSetTermios(unix.Stdin, unix.TCSETSF, &termios); err != nil {
		return fmt.Errorf("could not restore original console settings: %w", err)
	}
	return nil
}

//################################

```

In the `main()` function, we store the serialized version of the 
original `Termios` settings in byte slice called `origTermios`.
We can now `defer` a call to `restore()` passing `origTermios`
that will cause `restore()` to be called automatically when the
program exits, whether it exits by returning from `main()`, or
by calling the `exit()` function. 

Instead of calling `restore()` directly in defer, let us wrap it
in a [function clousure](https://tour.golang.org/moretypes/25) to
let us handle any errors that may happen during the call. Using defer 
and function clousures lets us avoid global variables.

| **Commit Title** | **File** |
|:-----------------|---------:|
| 6. Disable raw mode at exit| main.go|

```go
func main() {

    //######## Lines to Add/Change ##########

	origTermios, err := rawMode()
	if err != nil {
		fmt.Printf("Error: %s\n", err)
		os.Exit(1)
	}

	defer func() {
		if err := restore(origTermios); err != nil {
			fmt.Printf("Error: %s\n", err)
		}
    }()

	//################################

```


