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
`os.Stdin` which is of the type `os.File`. Since `os.File` implements
the standard `io.Reader` interface, we can simply wrap it up in a 
buffered i/o reader using `bufio.NewReader(os.Stdin)` which 
automatically buffers multiple key presses to be read at our
convenience and exposes convenient functions to read or peek a byte
at a time.

We then enter an infinite loop where we try to read data from the
standared input one byte at a time, check for errors or if we have
hit end of file marker (remember `os.Stdin` is an `os.File`) and 
then print what we just read.

When you compile and run `./gokilo`, your terminal gets hooked up to
the standard input, and so your keyboard input gets read into the `b` 
variable. However, by default your terminal starts in **canonical mode**, 
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
this chapter.

To exit the above program, press <kbd>Ctrl-D</kbd> to tell `read()` that it's
reached the end of file. Or you can always press <kbd>Ctrl-C</kbd> to signal
the process to terminate immediately.