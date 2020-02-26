# Entering Raw Mode

Let's try and read keypresses from the user. 

| **Commit Title** | **File** |
|:-----------------|---------:|
| Step 3: get keypresses from user| main.go|

```
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
		b, err := r.ReadByte()

		if err == io.EOF {
			break
		} else if err != nil {
			fmt.Printf("Error reading from keyboard: %s\n", err)
			os.Exit(1)
		}
		fmt.Printf(string(b))
	}
}
```
In Go, the standard input device is exposed in the os package as 
`os.Stdin`. Since `os.Stdin` is of the type `os.File` which implements
the standard `io.Reader` interface widely used across Go packages,
we can simply wrap it up in a Buffered IO reader by calling the function
`bufio.NewReader(os.Stdin)`. Buffered IO readers make it much easier
to 

Since `os.File` implements the `io.Reader` interface, we can easily
wrap this in a buffered IO reader  which makes it a lot easier to 
work with a stream of bytes 

Input in Go largely works using variants of the `io.Reader` interface.
The OS package exposes `Stdin` as an `io.Reader` and here we are wrapping
it in the Buffered IO reader through the `bufio.NewReader(os.Stdin)` function
call so that we can collect multiple key press codes in the buffer. As we 
will see later, some keys return more than one key-code

We then check 
as a reader in the `os` package. read()` and `STDIN_FILENO` come from `<unistd.h>`. We are asking `read()` to
read `1` byte from the standard input into the variable `c`, and to keep doing
it until there are no more bytes to read. `read()` returns the number of bytes
that it read, and will return `0` when it reaches the end of a file.

When you run `./kilo`, your terminal gets hooked up to the standard input, and
so your keyboard input gets read into the `c` variable. However, by default
your terminal starts in **canonical mode**, also called **cooked mode**. In
this mode, keyboard input is only sent to your program when the user presses
<kbd>Enter</kbd>. This is useful for many programs: it lets the user type in a
line of text, use <kbd>Backspace</kbd> to fix errors until they get their input
exactly the way they want it, and finally press <kbd>Enter</kbd> to send it to
the program. But it does not work well for programs with more complex user
interfaces, like text editors. We want to process each keypress as it comes in,
so we can respond to it immediately.

What we want is **raw mode**. Unfortunately, there is no simple switch you can
flip to set the terminal to raw mode. Raw mode is achieved by turning off a
great many flags in the terminal, which we will do gradually over the course of
this chapter.

To exit the above program, press <kbd>Ctrl-D</kbd> to tell `read()` that it's
reached the end of file. Or you can always press <kbd>Ctrl-C</kbd> to signal
the process to terminate immediately.