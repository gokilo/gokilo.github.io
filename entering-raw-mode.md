---
layout: default
title: Entering Raw Mode
nav_order: 3
---

# Entering Raw Mode

## 3. Get keypresses from user

Let's try to read keypress from the user. Create a file main.go and add the
following lines to read user input.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| Get keypress from user| main.go|

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
			fmt.Fprintf(os.Stderr, "Error: reading key from Stdin: %s\n", err)
			os.Exit(1)
		}
	}
}
```

Unix-like systems expose the keyboard device to programs abstracted as a file
called "Standard Input". This abstraction lets you easily do potentially useful
things like re-directing input from a disk file instead of the keyboard to
prorams & is often used in chaining command line tools.

Go exposes the standard input in the `os` package as an `os.File` typed package
level variable called `os.Stdin` . Since `os.File` implements the standard
`io.Reader` interface, we could potentially directly use the `File.Read()`
method to get keyboard input as if we were reading from a disk file. However, we
will take advantage of a buffered reader implementation provided by the Go
standard library's `bufio` that provides nice features like ability to read or
even "peek" a byte at a time. We call the `bufio.NewReader()` method with our
`os.Stdin` variable to get a buffered reader wrapping our keyboard input.

When you compile and run `./gokilo`, your terminal gets hooked up to the
standard input, and so your keyboard input gets read in the call to
`r.ReadByte()`. Since at this stage, we are not actually using the character we
read, we assign it to an `_` which throws away the return value as Go compiler
doesn't allow un-used variables.

In Go, errors are returned as values rather than as thrown exceptions etc. So
here we check for errors before continuing the infinite loop of reading from the
keyboard. Specifically, we check if the `err` variable is set to `io.EOF` which
marks the `end of file` marker and if so, we exit the program normally (remember
Stdin is a `os.File`, you could theoritically pipe from some other device to
it). If the error is not nil but some other value, we exit with an error message
& code. Note how we're using the `fmt.Fprintf()` function to write to the
**Standard Error** device provided by OS (usually the screen).

By default your terminal starts in **canonical mode**, also called **cooked
mode**. In this mode, keyboard input is only sent to your program when the user
presses <kbd>Enter</kbd>. This is useful for many programs: it lets the user
type in a line of text, use <kbd>Backspace</kbd> to fix errors until they get
their input exactly the way they want it, and finally press <kbd>Enter</kbd> to
send it to the program. But it does not work well for programs with more complex
user interfaces, like text editors. We want to process each keypress as it comes
in, so we can respond to it immediately.

What we want is **raw mode**. Unfortunately, there is no simple switch you can
flip to set the terminal to raw mode. Raw mode is achieved by turning off a
great many flags in the terminal, which we will do gradually over the course of
this chapter. This also works differently for Windows and Linux - so we will
also discuss how Go handles conditional compilation out of the box allowing you
to create a Windows version & a Linux version of the same function.

To exit the above program, press <kbd>Ctrl-D</kbd> to tell the program that it
has reached the end of file. Or you can always press <kbd>Ctrl-C</kbd> to signal
the process to terminate immediately.

## 4. Press <kbd>q</kbd> to quit?

To demonstrate how canonical mode or "cooked mode" works, we'll have the program
exit when it reads a <kbd>q</kbd> keypress from the user. From now on, code
changes between steps will be shown below in the "diff" format where the lines
marked with <kbd> -</kbd> should be deleted and lines marked with <kbd>+</kbd>
should be inserted.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| Press q to quit| main.go|

```diff
 func main() {
 
 	r := bufio.NewReader(os.Stdin)
 
 	for {
-		_, err := r.ReadByte()
+		b, err := r.ReadByte()
 
 		if err == io.EOF {
 			break
 		} else if err != nil {
 			fmt.Fprintf(os.Stderr, "Error: reading key from Stdin: %s\n", err)
 			os.Exit(1)
 		}
+
+		if b == 'q' {
+			break
+		}
 	}
 }
 ```

To quit this program, you will have to type a line of text that includes a `q`
in it, and then press enter. The program will quickly read the line of text one
character at a time until it reads the `q`, at which point the `for` loop will
terminate causing the program to exit. Any characters after the `q` will be left
unread on the input queue, and you may in some terminals see that fed into your
shell after your program exits.

This user experience of having to press <kbd>Enter</kbd> before any input gets
processed is obviously not useful in a text editor. 

## 5. Turn off echoing

To enter **raw mode** we have to interact with the operating system to request
it to turn off termial processing features like echoing keys we type to the
screen, waiting for <kbd>Enter</kbd> befor processing whatever is typed etc. so
that our Text Editor can control where characters appear on screen rather than
the operating system terminal.  

Go provides functions to interact with the operating systems through OS specific
packages under [`golang.org/x/sys`](https://godoc.org/golang.org/x/sys). In
Unix-like Operating Systems, we can can get terminal settings by using the
`unix.IoctlGetTermios()` function with `req` param set to `unix.TCGETS` from
[`golang.org/x/sys/unix`](https://godoc.org/golang.org/x/sys/unix) package. This
is equivalent to the Unix C Library function 
[`tcgetattr()`](https://linux.die.net/man/3/tcgetattr) from `termios.h`. To set
termial settings, we can similarly use `unix.IoctlSetTermios()` function with
`req` param set to `unix.TCSETSF` which is equivalent to 
[`tcsetattr()`](https://linux.die.net/man/3/tcsetattr). You may want to refer to
the C Library documentation of the equivalent functions as they're richer.

Our approach is going to be:
1. Read current terminal attributes with `IoctletTermios()` 
2. Modify the flags we want with binary bit manipulation routines
3. Write the modified attributes back using `IoctlSetTermios()` 

Let's first try turning off the `ECHO` feature this way. When we turn off the
`ECHO` feature, the screen will no longer show what key we are pressing. We will
create the function to carry out this operation in a new file in the same `main`
package called `rawmode_unix.go` to take advantage of a powerful feature of Go
called [build constraints](https://pkg.go.dev/cmd/go#hdr-Build_constraints). In
code below, take a look at the `// +build linux` in the first line which tells
the Go compiler to only compile this file when building for Linux. This way,
we can later port GoKilo to Windows by creating equivalent implementations in 
a `rawmode_windows.go` file containing the `// +build windows` constraints.

**Note**: If you're following the tutorial along other Unix-like OS's, you should 
modify the constraint to `// +build darwin` or `// +build freebsd` etc. Refer to
GOOS section in [build environment](https://golang.org/doc/install/source#environment).
While I have not personally tested this, it should work. 

| **Commit Title** | **Location** |
|:-----------------|---------:|
| Turn off echoing | rawmode_unix.go|

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
		return fmt.Errorf("rawMode: error getting terminal flags: %w", err)
	}

	termios.Lflag = termios.Lflag &^ unix.ECHO

	if err := unix.IoctlSetTermios(unix.Stdin, unix.TCSETSF, termios); err != nil {
		return fmt.Errorf("rawMode: error setting terminal flags: %w", err)
	}

	return nil
}

```

After you create the above file, run the folowing command in your terminal to
make sure Go modules will register the new `golang.org/x/sys/unix` dependency
correctly in the `go.mod` file and download it into your environment.
```
go mod tidy
```

Now, let's use the function in `main.go`

| **Commit Title** | **Location** |
|:-----------------|---------:|
| Turn off echoing | main.go|

```diff
 func main() {
 
+	if err := rawMode(); err != nil {
+		fmt.Fprintf(os.Stderr, "Error: %s\n", err)
+		os.Exit(1)
+	}
+
 	r := bufio.NewReader(os.Stdin)
 
 	for {
 		b, err := r.ReadByte()
 
 		if err == io.EOF {
 			break
 		} else if err != nil {
 			fmt.Fprintf(os.Stderr, "Error: reading key from Stdin: %s\n", err)
 			os.Exit(1)
 		}
 
 		if b == 'q' {
 			break
 		}
 	}
 }
```

The `unix.ECHO` feature causes each key you type to be printed to the terminal,
so you can see what you're typing. This is useful in canonical mode, but really
gets in the way when we are trying to carefully render a user interface in raw
mode. So we turn it off. This program does the same thing as the one in the
previous step, it just doesn't print what you are typing. You may be familiar
with this mode if you've ever had to type a password at the terminal, when using
`sudo` for example.

Terminal attributes can be read into a `Termios` struct by `IoctlGetTermios()`.
After modifying them, you can then apply them to the terminal using
`IoctlSetTermios()`. The `unix.TCSETFSF` (ie. Flush) argument specifies when to
apply the change: in this case, it waits for all pending output to be written to
the terminal, and also discards any input that hasn't been read.

The `Termios.Lflag` field is for "miscellaneous flags". The other flag fields
are `Termios.Iflag` (input flags), `Termios.Oflag` (output flags), and
`Termios.Cflag` (control flags), many of which we will have to modify to enable
raw mode.

`unix.ECHO` is a [bitflag](https://en.wikipedia.org/wiki/Bit_field), defined as
`00000000000000000000000000001000` in binary. We use the bitwise-NOT operator
(`~`) on this value to get `11111111111111111111111111110111`. We then
bitwise-AND this value with the flags field, which forces the fourth bit in the
flags field to become `0`, and causes every other bit to retain its current
value. Flipping bits like this is common in systems proramming.  We finally add
a call to this function in our `main()` function at the beginning.

When returning errors from `rawMode()` function, note how we're using the
`fmt.Errorf()` function with the `%w` formatting instruction to return an error
that **wraps** the error we receive from the library function with a bit of
context relevant messaging. To see the error in action, rather than building and
running `gokilo` from your terminal, try runing or debugging the program from
your code editor or IDE. You should see some kind of an error message like below
where the error from the library "inappropriate ioctl..." is wrapped by a more
contextual message explaining we ran into the issue in `rawMode()` function when
trying to get terminal flags.
```
Error: rawMode: error getting terminal flags: inappropriate ioctl for device
```
This error happens becuase during debugging, IDEs capture the standard
inputs and outputs provided to the program so that they can display any errors
or messages in the IDE. Therefore they no longer represent real physical devices
where you can do things like turn off `ECHO`. You'll see the same kind of error
if you run something like the below in your terminal where you're re-directing
input from the Unix `echo` command to `gokilo`. 
```
echo "abcd" | ./gokilo
```
You can even check by going back to the previous commit ("Press q to quit") and
rebuilding `gokilo` that this command above wouldn't have produced errors then
as the program would have happily received "abcd" from the standard input and would
have received <kbd>Ctrl-D</kbd> at the end of it.

After the program quits, depending on your shell, you may find your terminal is
still not echoing what you type. Don't worry, it will still listen to what you
type. Just press <kbd>Ctrl-C</kbd> to start a fresh line of input to your shell,
and type in `reset` and press <kbd>Enter</kbd>. This resets your terminal back
to normal in most cases. Failing that, you can always restart your terminal
emulator. We'll fix this whole problem in the next step.

## 6. Disable raw mode at exit

Let's be nice to the user and restore their terminal's original attributes when
our program exits. We'll save a copy of the `Termios` struct with its original
flags state and use `unix.IoctlSetTermios()` to restore it when the program
exits.

Go supports irst class functions, higher-order functions, user-defined function
types, function literals, closures and multiple return values. However, the way
we enter and exit raw mode related data we need to save differs by OS. To
improve OS portability and abstract away complexity from the rest of the program,
we'll have `rawMode()` return a simple function clousure can be invoked to
restore terminal settings.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| Disable raw mode at exit| rawmode_unix.go|

```diff
 // +build linux
 
 package main
 
 import (
 	"fmt"
+	"os"
 
 	"golang.org/x/sys/unix"
 )
 
-func rawMode() error {
+// rawMode modifies terminal settings to enable raw mode in a platform specific
+// way. It returns a function that can be invoked to restore previous settings.
+func rawMode() (func(), error) {
 
 	termios, err := unix.IoctlGetTermios(unix.Stdin, unix.TCGETS)
 	if err != nil {
-		return fmt.Errorf("rawMode: error getting terminal flags: %w", err)
+		return nil, fmt.Errorf("rawMode: error getting terminal flags: %w", err)
 	}
 
+	copy := *termios
+
 	termios.Lflag = termios.Lflag &^ unix.ECHO
 
 	if err := unix.IoctlSetTermios(unix.Stdin, unix.TCSETSF, termios); err != nil {
-		return fmt.Errorf("rawMode: error setting terminal flags: %w", err)
+		return nil, fmt.Errorf("rawMode: error setting terminal flags: %w", err)
 	}
 
-	return nil
+	return func() {
+		if err := unix.IoctlSetTermios(unix.Stdin, unix.TCSETSF, &copy); err != nil {
+			fmt.Fprintf(os.Stderr, "rawMode: error restoring originl console settings: %s", err)
+		}
+	}, nil
 }
```

Take a look at the new function signature of `rawMode()` which takes advantage
of multiple return value and first class functions support in Go.  `rawMode()`
returns two values now - a function that takes no parameters in addition to the
error value. The caller can save the returned function in a variable & invoke
it. We can define `rawMode()` function signature in exactly the same way across
all OSes abstracting away complexity from the program & enhancing portability.
```go 
func rawMode() (func(), error)
```

Also look at the last return statement. We're returning a function clousure that
can accesses the `copy` variable originally defined in this function even after
the function returns and use it to restore terminal settings. You can learn more
about Go's first class functions features in [this
codewalk](https://golang.org/doc/codewalk/functions/).
```go
	return func() {
		if err := unix.IoctlSetTermios(unix.Stdin, unix.TCSETSF, &copy); err != nil {
			fmt.Fprintf(os.Stderr, "rawMode: error restoring originl console settings: %s", err)
		}
	}, nil
```

Finally in the `main()` function, we store the returned function in the
`restoreFunc` variable and `defer` a call to it which will cause it to be
invoked whenever `main()` exits.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| Disable raw mode at exit| main.go|

```diff

 func main() {
 
-	if err := rawMode(); err != nil {
+	restoreFunc, err := rawMode()
+	if err != nil {
 		fmt.Fprintf(os.Stderr, "Error: %s\n", err)
 		os.Exit(1)
 	}
+	defer restoreFunc()
 
 	r := bufio.NewReader(os.Stdin)
 
 	for {
 		b, err := r.ReadByte()
 
 		if err == io.EOF {
 			break
 		} else if err != nil {
 			fmt.Fprintf(os.Stderr, "Error: reading key from Stdin: %s\n", err)
 			os.Exit(1)
 		}
 
 		if b == 'q' {
 			break
 		}
 	}
 }

```

## 7. Turn off canonical mode
There is an `ICANON` flag in `Termios` that allows us to turn off canonical
mode. This means we will finally be reading input byte-by-byte, instead of
line-by-line. Now the program will quit as soon as you press <kbd>q</kbd>. From
now on, we'll only show relevant lines of the changes vs. the whole listing.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| Turn off canonical mode | rawmode_unix.go|

```diff
 func rawMode() (func(), error) {
	
	// ...
 
 	copy := *termios
 
-	termios.Lflag = termios.Lflag &^ unix.ECHO
+	termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON)
 
 	if err := unix.IoctlSetTermios(unix.Stdin, unix.TCSETSF, termios); err != nil {
 		return nil, fmt.Errorf("rawMode: error setting terminal flags: %w", err)
 	}
	
	// ...
 }
```

## 8. Display keypresses

To get a better idea of how input in raw mode works, let's print out each byte
that we read. We'll print each character' s numeric ASCII value, as well as the
character it represents if it is a printable character.

A key choice we'll make at this stage is to add some level of basic support for
[Unicode UTF-8 encoding](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/)
as most modern languages and text documents support UTF-8 while it is also fully
backwards compatible with ASCII. However, we won't support advanced features like
[combining characters](https://en.wikipedia.org/wiki/Combining_character) etc.
Go has [support for Unicode](https://blog.golang.org/strings) in its
[standard library](https://pkg.go.dev/unicode) and a unicode code point is
represented by a data type called `rune`. So we will now use `ReadRune()`
instead of `ReadByte()`.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| Display Keypresses| main.go|

```diff
 package main
 
 import (
 	"bufio"
 	"fmt"
 	"io"
 	"os"
+	"unicode"
 )
 
 func main() {
 
 	restoreFunc, err := rawMode()
 	if err != nil {
 		fmt.Fprintf(os.Stderr, "Error: %s\n", err)
 		os.Exit(1)
 	}
 	defer restoreFunc()
 
 	r := bufio.NewReader(os.Stdin)
 
 	for {
-		b, err := r.ReadByte()
+		ru, _, err := r.ReadRune()
 
 		if err == io.EOF {
 			break
 		} else if err != nil {
 			fmt.Fprintf(os.Stderr, "Error: reading key from Stdin: %s\n", err)
 			os.Exit(1)
 		}
+		if unicode.IsControl(ru) {
+			fmt.Printf("%d\n", ru)
+		} else {
+			fmt.Printf("%d (%c)\n", ru, ru)
+		}
 
-		if b == 'q' {
+		if ru == 'q' {
 			break
 		}
 	}
 }
```

We are using `unicode.IsControl()` to identify if the character is a regular
printable character or a control character. For printable characters, we'll
display both its character code as a number & the character itself using
`fmt.Printf()` format instructions. For control characters, we'll only display
the numerical character code.
- `%d` tells it to format the `rune` as a decimal number (character code)
- `%c` tells it to display the printable character of the `rune`

This is a very useful program. It shows us how various keypresses translate into
the bytes we read. Most ordinary keys translate directly into the characters
they represent. But try seeing what happens when you press the arrow keys, or
<kbd>Escape</kbd>, or <kbd>Page Up</kbd>, or <kbd>Page Down</kbd>, or
<kbd>Home</kbd>, or <kbd>End</kbd>, or <kbd>Backspace</kbd>, or
<kbd>Delete</kbd>, or <kbd>Enter</kbd>. Try key combinations with
<kbd>Ctrl</kbd>, like <kbd>Ctrl-A</kbd>, <kbd>Ctrl-B</kbd>, etc. You'll notice a
few interesting things:

- Movement keys like Arrow keys, <kbd>Page Up</kbd>, <kbd>Page Down</kbd>,
  <kbd>Home</kbd>, and <kbd>End</kbd> all input 3 or 4 characters to the
  terminal: `27`, `'['`, and then one or two more characters. This is known as
  an *escape sequence*. All escape sequences start with a `27` byte. Pressing
  <kbd>Escape</kbd> sends a single `27` byte as input.
- Function keys also return escape sequences
- <kbd>Backspace</kbd> is byte `127` while <kbd>Delete</kbd> is a 4-byte escape
  sequence.
- <kbd>Enter</kbd> is byte `10`, which is a newline character, also known as
  `'\n'`.
- <kbd>Ctrl-A</kbd> is `1`, <kbd>Ctrl-B</kbd> is `2`, <kbd>Ctrl-C</kbd> is...
  oh, that terminates the program, right. But the <kbd>Ctrl</kbd> key
  combinations that do work seem to map the letters A&ndash;Z to the codes
  1&ndash;26.
- On some consoles, if you happen to press <kbd>Ctrl-S</kbd>, you may find your
  program seems to be frozen. What you've done is you've asked your program to
  [stop sending you output](https://en.wikipedia.org/wiki/Software_flow_control). Press
  <kbd>Ctrl-Q</kbd> to tell it to resume sending you output.
- Also, if you press <kbd>Ctrl-Z</kbd> (or maybe <kbd>Ctrl-Y</kbd>), your
  program will be suspended to the background. Run the `fg` command to bring it
  back to the foreground but it may no longer be in raw mode. It may also quit
  immediately after you do that, as a result of the underlying File reader
  returning error.
- You should be able to [input
  unicode](https://en.wikipedia.org/wiki/Unicode_input) characters that are not
  in your keyboard either by copy/paste or one of the supported unicode input
  mechanisms in your OS.

## 9. Turn off <kbd>Ctrl-C</kbd> and <kbd>Ctrl-Z</kbd> signals

By default, <kbd>Ctrl-C</kbd> sends a `SIGINT` signal to the current process
which causes it to terminate, and <kbd> Ctrl-Z</kbd> sends a `SIGTSTP` signal to
the current process which causes it to suspend. Let's turn off the sending of
both of these signals using the `unix.ISIG` flag

| **Commit Title** | **Location** |
|:-----------------|---------:|
| Turn off Ctrl-C and Ctrl-Z signals| rawmode_unix.go|

```diff
 // ...
 func rawMode() (func(), error) {

    // ...
 
    copy := *termios
 
-   termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON)
+   termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON | unix.ISIG)
 
    if err := unix.IoctlSetTermios(unix.Stdin, unix.TCSETSF, termios); err != nil {
        return nil, fmt.Errorf("rawMode: error setting terminal flags: %w", err)
    }

    // ...
 
 }
```

Now <kbd>Ctrl-C</kbd> can be read as a `3` byte and <kbd>Ctrl-Z</kbd> can be
read as a `26` byte.  This also disables <kbd>Ctrl-Y</kbd> on macOS, which is
like <kbd>Ctrl-Z</kbd> except it waits for the program to read input before
suspending it.

## 10. Disable <kbd>Ctrl-S</kbd> and <kbd>Ctrl-Q</kbd>

By default, <kbd>Ctrl-S</kbd> and <kbd>Ctrl-Q</kbd> are used for 
[software flow control](https://en.wikipedia.org/wiki/Software_flow_control).
<kbd>Ctrl-S</kbd> stops data from being transmitted to the terminal until you
press <kbd>Ctrl-Q</kbd>. This originates in the days when you might want to
pause the transmission of data to let a device like a printer catch up. Let's
just turn off that feature.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| Disable Ctrl-S and Ctrl-Q | rawmode_unix.go|

```diff
 
 // ...

 func rawMode() (func(), error) {
 
	// ...
 
 	copy := *termios
 
 	termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON | unix.ISIG)
+	termios.Iflag = termios.Iflag &^ (unix.IXON)
 
 	if err := unix.IoctlSetTermios(unix.Stdin, unix.TCSETSF, termios); err != nil {
 		return nil, fmt.Errorf("rawMode: error setting terminal flags: %w", err)
 	}
 
	// ...
 }

```

The `I` in `IXON` stands for "input flag" (which it is, unlike the other `L`
flags we've seen so far) and `XON` comes from the names of the two control
characters that <kbd>Ctrl-S</kbd> and <kbd>Ctrl-Q</kbd> produce: `XOFF` to pause
transmission and `XON` to resume transmission.

Now <kbd>Ctrl-S</kbd> can be read as a `19` byte and <kbd>Ctrl-Q</kbd> can be
read as a `17` byte.

## 11. Disable <kbd>Ctrl-V</kbd>

On some systems, when you type <kbd>Ctrl-V</kbd>, the terminal waits for you to
type another character and then sends that character literally. For example,
before we disabled <kbd>Ctrl-C</kbd>, you might've been able to type
<kbd>Ctrl-V</kbd> and then <kbd>Ctrl-C</kbd> to input a `3` byte. We can turn
off this feature using the `IEXTEN` flag.

Turning off `IEXTEN` also fixes <kbd>Ctrl-O</kbd> in macOS, whose terminal
driver is otherwise set to discard that control character. It is another flag
that starts with `I` but actually belongs in the `Lflag` field.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| Disable Ctrl-V | rawmode_unix.go|

```diff
 
 // ...

 func rawMode() (func(), error) {
 
	// ...
 
 	copy := *termios
 
-	termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON | unix.ISIG)
+	termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON | unix.ISIG | unix.IEXTEN)
 	termios.Iflag = termios.Iflag &^ (unix.IXON)
 
 	if err := unix.IoctlSetTermios(unix.Stdin, unix.TCSETSF, termios); err != nil {
 		return nil, fmt.Errorf("rawMode: error setting terminal flags: %w", err)
 	}
 
	// ...
 }
```

<kbd>Ctrl-V</kbd> can now be read as a `22` byte, and <kbd>Ctrl-O</kbd> as a
`15` byte.

## 12. Fix <kbd>Ctrl-M</kbd>

If you run the program now and go through the whole alphabet while holding down
<kbd>Ctrl</kbd>, you should see that we have every letter except <kbd>M</kbd>.
<kbd>Ctrl-M</kbd> is weird: it's being read as `10`, when we expect it to be
read as `13`, since it is the 13th letter of the alphabet, and <kbd>Ctrl-J</kbd>
already produces a `10`. What else produces `10`? The <kbd>Enter</kbd> key does.
It turns out that the terminal is helpfully translating any carriage returns
(`13`, `'\r'`) inputted by the user into newlines (`10`, `'\n'`). Let's turn off
this feature.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| Fix Ctrl-M | rawmode_unix.go|

```diff
 
 // ...

 func rawMode() (func(), error) {

	// ...
 
 	copy := *termios
 
 	termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON | unix.ISIG | unix.IEXTEN)
-	termios.Iflag = termios.Iflag &^ (unix.IXON)
+	termios.Iflag = termios.Iflag &^ (unix.IXON | unix.ICRNL)
 
 	if err := unix.IoctlSetTermios(unix.Stdin, unix.TCSETSF, termios); err != nil {
 		return nil, fmt.Errorf("rawMode: error setting terminal flags: %w", err)
 	}
 
    // ...
 }
```

The `I` in `ICRNL` stands for "input flag", `CR` stands for "carriage return",
and `NL` stands for "new line".  Now <kbd>Ctrl-M</kbd> is read as a `13`
(carriage return), and the <kbd>Enter</kbd> key is also read as a `13`.

## 13. Turn off all output processing

It turns out that the terminal does a similar translation on the output side. It
translates each newline (`"\n"`) we print into a carriage return followed by a
newline (`"\r\n"`). The terminal requires both of these characters in order to
start a new line of text. The carriage return moves the cursor back to the
beginning of the current line, and the newline moves the cursor down a line,
scrolling the screen if necessary. (These two distinct operations originated in
the days of typewriters and
[teletypes](https://en.wikipedia.org/wiki/Teleprinter).)

We will turn off all output processing features by turning off the `OPOST` flag.
In practice, the `"\n"` to `"\r\n"` translation is likely the only output
processing feature turned on by default. `O` means it's an output flag, and I
assume `POST` stands for "post-processing of output".

| **Commit Title** | **Location** |
|:-----------------|---------:|
| Turn off all output processing | rawmode_unix.go|

```diff
 
 // ...

 func rawMode() (func(), error) {

	// ...
 
 	copy := *termios
 
 	termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON | unix.ISIG | unix.IEXTEN)
	termios.Iflag = termios.Iflag &^ (unix.IXON | unix.ICRNL)
+   termios.Oflag = termios.Oflag &^ (unix.OPOST)
 
 	if err := unix.IoctlSetTermios(unix.Stdin, unix.TCSETSF, termios); err != nil {
 		return nil, fmt.Errorf("rawMode: error setting terminal flags: %w", err)
 	}
 
    // ...
 }
```

## 14. Fix carriage returns

If you ran the program after turning off `OPOST`, you'll see that the newline
characters we're printing are only moving the cursor down, and not to the left
side of the screen. To fix that, let's add carriage returns to our
`fmt.Printf()` statements.  From now on, we'll have to write out the full
`"\r\n"` whenever we want to start a new line.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| Fix carriage returns | main.go|

```diff

 // ..

 func main() {
 
 	restoreFunc, err := rawMode()
 	if err != nil {
-		fmt.Fprintf(os.Stderr, "Error: %s\n", err)
+		fmt.Fprintf(os.Stderr, "Error: %s\r\n", err)
 		os.Exit(1)
 	}
 	defer restoreFunc()
 
 	r := bufio.NewReader(os.Stdin)
 
 	for {
 		ru, _, err := r.ReadRune()
 
 		if err == io.EOF {
 			break
 		} else if err != nil {
-			fmt.Fprintf(os.Stderr, "Error: reading key from Stdin: %s\n", err)
+			fmt.Fprintf(os.Stderr, "Error: reading key from Stdin: %s\r\n", err)
 			os.Exit(1)
 		}
 		if unicode.IsControl(ru) {
-			fmt.Printf("%d\n", ru)
+			fmt.Printf("%d\r\n", ru)
 		} else {
-			fmt.Printf("%d (%c)\n", ru, ru)
+			fmt.Printf("%d (%c)\r\n", ru, ru)
 		}
 
 		if ru == 'q' {
 			break
 		}
 	}
 }
```

| **Commit Title** | **Location** |
|:-----------------|---------:|
| Fix carriage returns | rawmode_unix.go|

```diff

 // ...

 func rawMode() (func(), error) {
 
	// ...
 
 	return func() {
 		if err := unix.IoctlSetTermios(unix.Stdin, unix.TCSETSF, &copy); err != nil {
-			fmt.Fprintf(os.Stderr, "rawMode: error restoring originl console settings: %s", err)
+			fmt.Fprintf(os.Stderr, "rawMode: error restoring originl console settings: %s\r\n", err)
 		}
 	}, nil
 }
```
## 15. Turn off Miscellaneous flags

Let's turn off a few more flags. `BRKINT`, `INPCK`, `ISTRIP`, and `CS8`.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| Turn off Miscellaneous flags | main.go|

```diff

 // ...

 func rawMode() (func(), error) {
 
	// ...
 
 	copy := *termios
 
 	termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON | unix.ISIG | unix.IEXTEN)
-	termios.Iflag = termios.Iflag &^ (unix.IXON | unix.ICRNL)
+	termios.Iflag = termios.Iflag &^ (unix.IXON | unix.ICRNL | unix.BRKINT | unix.INPCK | unix.ISTRIP)
 	termios.Oflag = termios.Oflag &^ (unix.OPOST)
+	termios.Cflag = termios.Cflag | unix.CS8
 
 	if err := unix.IoctlSetTermios(unix.Stdin, unix.TCSETSF, termios); err != nil {
 		return nil, fmt.Errorf("rawMode: error setting terminal flags: %w", err)
 	}

	// ...
 }
```

This step probably won't have any observable effect for you, because these flags
are either already turned off, or they don't really apply to modern terminal
emulators. But at one time or another, switching them off was considered (by
someone) to be part of enabling "raw mode", so we carry on the tradition (of
whoever that someone was) in our program. From what we can tell

- When `BRKINT` is turned on, a [break
  condition](https://www.cmrr.umn.edu/~strupp/serial.html#2_3_3) will cause a
  `SIGINT` signal to be sent to the program, like pressing `Ctrl-C`.
- `INPCK` enables parity checking, which doesn't seem to apply to modern
  terminal emulators.
- `ISTRIP` causes the 8th bit of each input byte to be stripped, meaning it will
  set it to `0`. This is probably already turned off.
- `CS8` is not a flag, it is a bit mask with multiple bits, which we set using
  the bitwise-OR (`|`) operator unlike all the flags we are turning off. It sets
  the character size (CS) to 8 bits per byte. On my system, it's already set
  that way.

## 16. Safe Exit

Now that we are able to get into raw mode and also restore safely, let's create
a helper function that we can call anywhere from our program to exit raw mode
safely, show error messages if any and indicate to the OS whether we exited
cleanly or with errors.

To do this, we are agian going to take advantage of 
[First Class Functions](https://golang.org/doc/codewalk/functions/). 
We will first create a globally accessible variable `safeExit` which takes the
type `func(error)`. In other words, we can assign to this variable any function
that takes an `error` as a parameter and returns nothing.

Then after we get `restoreFunc` by calling `rawMode()`, we can set `safeExit` to
an anonymous function that restores terminal settings and also prints an error
message if `safeExit` was called with a non-`nil` error value.

Finally, we can simply change the `defer` statement to call `safeExit(nil)` to
call it with no error and similarly change the places we need to exit with error
to `safeExit(err)`

| **Commit Title** | **Location** |
|:-----------------|---------:|
| Safe Exit | main.go|

```diff
 package main
 
 import (
 	"bufio"
 	"fmt"
 	"io"
 	"os"
 	"unicode"
 )
 
+var safeExit func(error)
+
 func main() {
 
 	restoreFunc, err := rawMode()
+
+	safeExit = func(err error) {
+		if restoreFunc != nil {
+			restoreFunc()
+		}
+
+		if err != nil {
+			fmt.Fprintf(os.Stderr, "Error: %s\r\n", err)
+			os.Exit(1)
+		}
+		os.Exit(0)
+	}
+	defer safeExit(nil)
+
 	if err != nil {
-		fmt.Fprintf(os.Stderr, "Error: %s\r\n", err)
-		os.Exit(1)
+		safeExit(err)
 	}
-	defer restoreFunc()
 
 	r := bufio.NewReader(os.Stdin)
 
 	for {
 		ru, _, err := r.ReadRune()
 
 		if err == io.EOF {
 			break
 		} else if err != nil {
-			fmt.Fprintf(os.Stderr, "Error: reading key from Stdin: %s\r\n", err)
-			os.Exit(1)
+			safeExit(err)
 		}
 		if unicode.IsControl(ru) {
 			fmt.Printf("%d\r\n", ru)
 		} else {
 			fmt.Printf("%d (%c)\r\n", ru, ru)
 		}
 
 		if ru == 'q' {
 			break
 		}
 	}
 }

```

That just about concludes this chapter on entering raw mode. In the [next
chapter](/raw-input-and-output.html), we'll do some more low-level terminal
input/output handling, and use that to draw to the screen and allow the user to
move the cursor around.
