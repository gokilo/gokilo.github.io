# Entering Raw Mode

Let's try and read keypresses from the user. (The lines you need to add are
highlighted and marked with arrows.)

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
		r, _, err := r.ReadRune()

		if err == io.EOF {
			break
		} else if err != nil {
			fmt.Printf("Error reading from keyboard: %s\n", err)
			os.Exit(1)
		}
		fmt.Printf(string(r))
	}
}
```

`read()` and `STDIN_FILENO` come from `<unistd.h>`. We are asking `read()` to
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