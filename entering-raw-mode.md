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

In Go, the standard input device is exposed in the os package as 
package level `os.File` type variable called `os.Stdin`. Rather
than directly reading from the file C-style, we will prefer to
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

		// ***** Lines to Add
		if b == 'q'{
			break
		}
		// *****
	}
}
```

To quit this program, you will have to type a line of text that includes a `q`
in it, and then press enter. The program will quickly read the line of text one
character at a time until it reads the `q`, at which point the `for` loop
will stop and the program will exit. Any characters after the `q` will be left
unread on the input queue, and you may see that input being fed into your shell
after your program exits.