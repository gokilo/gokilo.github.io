---
layout: default
title: Entering Raw Mode
nav_order: 3
---

# Entering Raw Mode

## 3. Get keypresses from user

Let's try to read keypress from the user. Create a file main.go and add the following lines to read user input.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| get keypress from user| main.go|

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

Unix-like systems expose the keyboard device to programs abstracted as a file called "Standard Input". This abstraction
lets you easily do potentially useful things like re-directing input from a disk file instead of the keyboard to prorams
& is often used in chaining command line tools.

Go exposes the standard input in the `os` package as an `os.File` typed package level variable called `os.Stdin` .
Since `os.File` implements the standard `io.Reader` interface, we could potentially directly use the `File.Read()`
method to get keyboard input as if we were reading from a disk file. However, we will take advantage of a buffered
reader implementation provided by the Go standard library's `bufio` that provides nice features like ability to read or
even "peek" a byte at a time. We call the `bufio.NewReader()`
method with our `os.Stdin` variable to get a buffered reader wrapping our keyboard input.

When you compile and run `./gokilo`, your terminal gets hooked up to the standard input, and so your keyboard input gets
read in the call to `r.ReadByte()`. Since at this stage, we are not actually using the character we read, we assign it
to an `_` which throws away the return value as Go compiler doesn't allow un-used variables.

In Go, errors are returned as values rather than as thrown exceptions etc. So here we check for errors before continuing
the infinite loop of reading from the keyboard. Specifically, we check if the `err` variable is set to `io.EOF` which
marks the `end of file` marker and if so, we exit the program normally (remember Stdin is a `os.File`, you could
theoritically pipe from some other device to it). If the error is not nil but some other value, we exit with an error
message & code.

By default your terminal starts in **canonical mode**, also called **cooked mode**. In this mode, keyboard input is only
sent to your program when the user presses <kbd>Enter</kbd>. This is useful for many programs: it lets the user type in
a line of text, use <kbd>Backspace</kbd>
to fix errors until they get their input exactly the way they want it, and finally press <kbd>Enter</kbd> to send it to
the program. But it does not work well for programs with more complex user interfaces, like text editors. We want to
process each keypress as it comes in, so we can respond to it immediately.

What we want is **raw mode**. Unfortunately, there is no simple switch you can flip to set the terminal to raw mode. Raw
mode is achieved by turning off a great many flags in the terminal, which we will do gradually over the course of this
chapter. This also works differently for Windows and Linux - so we will also discuss how Go handles conditional
compilation out of the box allowing you to create a Windows version & a Linux version of the same function.

To exit the above program, press <kbd>Ctrl-D</kbd> to tell the program that it has reached the end of file. Or you can
always press <kbd>Ctrl-C</kbd> to signal the process to terminate immediately.

## 4. Press <kbd>q</kbd> to quit?

To demonstrate how canonical mode works, we'll have the program exit when it reads a <kbd>q</kbd> keypress from the
user. From now on, code changes between steps will be shown below in the "diff" format where the lines marked with <kbd>
-</kbd>
should be deleted and lines marked with <kbd>+</kbd> should be inserted.

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
 			fmt.Printf("Error reading from Stdin: %s\n", err)
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
in it, and then press enter. The program will quickly read the line of text one character at a time until it reads
the `q`, at which point the `for` loop will terminate causing the program to exit. Any characters after the `q` will be
left unread on the input queue, and you may in some terminals see that fed into your shell after your program exits.

This user experience of having to press <kbd>Enter</kbd> before any input gets processed is obviously not useful in 
a text editor. In the rest of this section of the tutorial, we will use functions provided by Go to interact with
the operating system and change terminal settings to support better user interaction.

## 5. Turn off echoing

Go provides functions to interact at a low level interaction operating systems OS specific packages under
the [`golang.org/x/sys`](https://godoc.org/golang.org/x/sys)
namespace . To get and control terminal attributes in Unix, we can use the `IoctlGetTermios()` and `IoctlSetTermios()`
functions from the `unix` package which mirror the more traditional `tcgetattr()`
and `tcsetattr()` functions in the Unix C library.

We can set a terminal's attributes by

1. Reading the current terminal attributes into a `unix.Termios` struct
2. Modifying the value of one or more fields in the struct with bit manipulation
3. Writing back out the modified terminal attributes.

Let's try turning off the `ECHO` feature this way. We will create the function to carry out this operation in a new file
in the same `main`
package called `rawmode_unix.go` to take advantage of a powerful feature of go called build tags. In code below, take a
look at the very first line
`// +build linux` which tells the Go compiler to only use this file when building for Linux targets. This way, we can
later port GoKilo to Windows by creating a platform appropriate implementation with `// +build windows` tag

**Note**: you may use `darwin` or `freebsd` etc. instead of `linux` if you're following the tutorial on those systems. I
haven't tested it personally, but it should work.

| **Commit Title** | **Location** |
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

	return nil
}

```

| **Commit Title** | **Location** |
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

The `unix.ECHO` feature causes each key you type to be printed to the terminal, so you can see what you're typing. This
is useful in canonical mode, but really gets in the way when we are trying to carefully render a user interface in raw
mode. So we turn it off. This program does the same thing as the one in the previous step, it just doesn't print what
you are typing. You may be familiar with this mode if you've ever had to type a password at the terminal, when
using `sudo` for example.

Terminal attributes can be read into a `Termios` struct by `IoctlGetTermios()`. After modifying them, you can then apply
them to the terminal using
`IoctlSetTermios()`. The `unix.TCSETFSF` (ie. Flush) argument specifies when to apply the change: in this case, it waits
for all pending output to be written to the terminal, and also discards any input that hasn't been read.

The `Termios.Lflag` field is for "miscellaneous flags". The other flag fields are `Termios.Iflag` (input flags)
, `Termios.Oflag` (output flags), and
`Termios.Cflag` (control flags), most of which we will have to modify to enable raw mode.

`unix.ECHO` is a [bitflag](https://en.wikipedia.org/wiki/Bit_field), defined as
`00000000000000000000000000001000` in binary. We use the bitwise-NOT operator
(`~`) on this value to get `11111111111111111111111111110111`. We then bitwise-AND this value with the flags field,
which forces the fourth bit in the flags field to become `0`, and causes every other bit to retain its current value.
Flipping bits like this is common in systems proramming.

We finally add a call to this function in our `main()` function at the beginning.

After the program quits, depending on your shell, you may find your terminal is still not echoing what you type. Don't
worry, it will still listen to what you type. Just press <kbd>Ctrl-C</kbd> to start a fresh line of input to your shell,
and type in `reset` and press <kbd>Enter</kbd>. This resets your terminal back to normal in most cases. Failing that,
you can always restart your terminal emulator. We'll fix this whole problem in the next step.

## 6. Disable raw mode at exit

Let's be nice to the user and restore their terminal's original attributes when our program exits. We'll save a copy of
the `Termios` struct in its original state, and use `unix.IoctlSetTermios()` to apply it to the terminal when the
program exits with a new function called `restore()`.

Keeping platform portability in mind, `Termios` is not really defined on the Windows platform, so rather than have
the `rawMode()` function return it directly, we will serialize it using Go's native `encoding/gob`
package into a byte slice that can be saved by he caller and passed back to be decoded and restored. Since Go supports
multiple return values from functions, we can return both the byte slice and any error encountered.

| **Commit Title** | **Location** |
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

In the `main()` function, we store the serialized version of the original `Termios` settings in byte slice
called `origTermios`. We can now `defer` a call to `restore()` passing `origTermios`
that will cause `restore()` to be called automatically when the program exits, whether it exits by returning
from `main()`, or by calling the `exit()` function.

Also instead of calling `restore()` directly in defer, let us wrap it in
a [function clousure](https://tour.golang.org/moretypes/25) to let us handle any errors that may happen during the call.
Using defer and function clousures lets us avoid global variables.

| **Commit Title** | **Location** |
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

defer func () {
if err := restore(origTermios); err != nil {
fmt.Printf("Error: %s\n", err)
}
}()

//################################

```

## 7. Turn off canonical mode

There is an `ICANON` flag that allows us to turn off canonical mode. This means we will finally be reading input
byte-by-byte, instead of line-by-line. Now the program will quit as soon as you press <kbd>q</kbd>.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| 7. Turn off canonical mode| rawmode_unix.go|

```go
        return nil, fmt.Errorf("could not serialize console settings: %w", err)
}

//######## Lines to Add/Change ##########
termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON)
//################################

if err := unix.IoctlSetTermios(unix.Stdin, unix.TCSETSF, termios); err != nil {
```

## 8. Display keypresses

To get a better idea of how input in raw mode works, let's print out each byte that we read. We'll print each character'
s numeric ASCII value, as well as the character it represents if it is a printable character.

Let's first create a small function `isCntrl()` to tests whether a character is a control character. Control characters
are nonprintable characters that we don't want to print to the screen. ASCII codes 0&ndash;31 are all control
characters, and 127 is also a control character. ASCII codes 32&ndash;126 are all printable. (Check out the
[ASCII table](http://asciitable.com) to see all of the characters.)

| **Commit Title** | **Location** |
|:-----------------|---------:|
| 8. Display Keypresses| main.go|

```go

"os"
)

//######## Lines to Add/Change ##########

func isCntrl(b byte) bool {
if b <= 0x1f || b == 0x7f {
return true
}
return false
}

//################################

func main() {

```

`fmt.Printf()` can print multiple representations of a byte. `%d` tells it to format the byte as a decimal number (its
ASCII code), and `%c` tells it to write out the byte directly, as a character.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| 8. Display Keypresses| main.go|

```go

fmt.Printf("Error reading from Stdin: %s\n", err)
os.Exit(1)
}

//######## Lines to Add/Change ##########

if isCntrl(b) {
fmt.Printf("%d\n", b)
} else {
fmt.Printf("%d (%c)\n", b, b)
}

//################################

if b == 'q' {

```

This is a very useful program. It shows us how various keypresses translate into the bytes we read. Most ordinary keys
translate directly into the characters they represent. But try seeing what happens when you press the arrow keys,
or <kbd>Escape</kbd>, or <kbd>Page Up</kbd>, or <kbd>Page Down</kbd>, or
<kbd>Home</kbd>, or <kbd>End</kbd>, or <kbd>Backspace</kbd>, or
<kbd>Delete</kbd>, or <kbd>Enter</kbd>. Try key combinations with
<kbd>Ctrl</kbd>, like <kbd>Ctrl-A</kbd>, <kbd>Ctrl-B</kbd>, etc.

You'll notice a few interesting things:

* Arrow keys, <kbd>Page Up</kbd>, <kbd>Page Down</kbd>, <kbd>Home</kbd>, and
  <kbd>End</kbd> all input 3 or 4 bytes to the terminal: `27`, `'['`, and then one or two other characters. This is
  known as an *escape sequence*. All escape sequences start with a `27` byte. Pressing <kbd>Escape</kbd> sends a
  single `27` byte as input.
* <kbd>Backspace</kbd> is byte `127`. <kbd>Delete</kbd> is a 4-byte escape sequence.
* <kbd>Enter</kbd> is byte `10`, which is a newline character, also known as
  `'\n'`.
* <kbd>Ctrl-A</kbd> is `1`, <kbd>Ctrl-B</kbd> is `2`, <kbd>Ctrl-C</kbd> is... oh, that terminates the program, right.
  But the <kbd>Ctrl</kbd> key combinations that do work seem to map the letters A&ndash;Z to the codes 1&ndash;26.

On some consoles, if you happen to press <kbd>Ctrl-S</kbd>, you may find your program seems to be frozen. What you've
done is you've asked your program to [stop sending you output](https://en.wikipedia.org/wiki/Software_flow_control).
Press
<kbd>Ctrl-Q</kbd> to tell it to resume sending you output.

Also, if you press <kbd>Ctrl-Z</kbd> (or maybe <kbd>Ctrl-Y</kbd>), your program will be suspended to the background. Run
the `fg` command to bring it back to the foreground but it may no longer be in raw mode. It may also quit immediately
after you do that, as a result of the underlying File reader returning error.

## 9. Turn off <kbd>Ctrl-C</kbd> and <kbd>Ctrl-Z</kbd> signals

By default, <kbd>Ctrl-C</kbd> sends a `SIGINT` signal to the current process which causes it to terminate, and <kbd>
Ctrl-Z</kbd> sends a `SIGTSTP` signal to the current process which causes it to suspend. Let's turn off the sending of
both of these signals.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| 9. Turn off Ctrl-C and Ctrl-Z signals| rawmode_unix.go|

```go

//######## Lines to Add/Change ##########

termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON | unix.ISIG)

//################################
```

`ISIG` comes from `<termios.h>`. Like `ICANON`, it starts with `I` but isn't an input flag.

Now <kbd>Ctrl-C</kbd> can be read as a `3` byte and <kbd>Ctrl-Z</kbd> can be read as a `26` byte.

This also disables <kbd>Ctrl-Y</kbd> on macOS, which is like <kbd>Ctrl-Z</kbd>
except it waits for the program to read input before suspending it.

## 10. Disable <kbd>Ctrl-S</kbd> and <kbd>Ctrl-Q</kbd>

By default, <kbd>Ctrl-S</kbd> and <kbd>Ctrl-Q</kbd> are used for
[software flow control](https://en.wikipedia.org/wiki/Software_flow_control).
<kbd>Ctrl-S</kbd> stops data from being transmitted to the terminal until you press <kbd>Ctrl-Q</kbd>. This originates
in the days when you might want to pause the transmission of data to let a device like a printer catch up. Let's just
turn off that feature.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| 10. Disable Ctrl-S and Ctrl-Q | rawmode_unix.go|

```go


termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON | unix.ISIG)

//######## Lines to Add/Change ##########
termios.Iflag = termios.Iflag &^ (unix.IXON)
//################################

```

The `I` in `IXON` stands for "input flag" (which it is, unlike the other `L` flags we've seen so far) and `XON` comes
from the names of the two control characters that <kbd>Ctrl-S</kbd> and <kbd>Ctrl-Q</kbd>
produce: `XOFF` to pause transmission and `XON` to resume transmission.

Now <kbd>Ctrl-S</kbd> can be read as a `19` byte and <kbd>Ctrl-Q</kbd> can be read as a `17` byte.

## 11. Disable <kbd>Ctrl-V</kbd>

On some systems, when you type <kbd>Ctrl-V</kbd>, the terminal waits for you to type another character and then sends
that character literally. For example, before we disabled <kbd>Ctrl-C</kbd>, you might've been able to type
<kbd>Ctrl-V</kbd> and then <kbd>Ctrl-C</kbd> to input a `3` byte. We can turn off this feature using the `IEXTEN` flag.

Turning off `IEXTEN` also fixes <kbd>Ctrl-O</kbd> in macOS, whose terminal driver is otherwise set to discard that
control character. It is another flag that starts with `I` but actually belongs in the `Lflag` field.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| 11. Disable Ctrl-V | rawmode_unix.go|

```go

//######## Lines to Add/Change ##########
termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON | unix.ISIG | unix.IEXTEN)
//################################
termios.Iflag = termios.Iflag &^ (unix.IXON)

```

<kbd>Ctrl-V</kbd> can now be read as a `22` byte, and <kbd>Ctrl-O</kbd> as a
`15` byte.

## 12. Fix <kbd>Ctrl-M</kbd>

If you run the program now and go through the whole alphabet while holding down
<kbd>Ctrl</kbd>, you should see that we have every letter except <kbd>M</kbd>.
<kbd>Ctrl-M</kbd> is weird: it's being read as `10`, when we expect it to be read as `13`, since it is the 13th letter
of the alphabet, and
<kbd>Ctrl-J</kbd> already produces a `10`. What else produces `10`? The
<kbd>Enter</kbd> key does.

It turns out that the terminal is helpfully translating any carriage returns
(`13`, `'\r'`) inputted by the user into newlines (`10`, `'\n'`). Let's turn off this feature.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| 12. Fix Ctrl-M | rawmode_unix.go|

```go

termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON | unix.ISIG | unix.IEXTEN)
//######## Lines to Add/Change ##########
termios.Iflag = termios.Iflag &^ (unix.IXON | unix.ICRNL)
//################################

```

The `I` in `ICRNL` stands for "input flag", `CR` stands for "carriage return", and `NL` stands for "new line".

Now <kbd>Ctrl-M</kbd> is read as a `13` (carriage return), and the
<kbd>Enter</kbd> key is also read as a `13`.

## 13. Turn off all output processing

It turns out that the terminal does a similar translation on the output side. It translates each newline (`"\n"`) we
print into a carriage return followed by a newline (`"\r\n"`). The terminal requires both of these characters in order
to start a new line of text. The carriage return moves the cursor back to the beginning of the current line, and the
newline moves the cursor down a line, scrolling the screen if necessary. (These two distinct operations originated in
the days of typewriters and
[teletypes](https://en.wikipedia.org/wiki/Teleprinter).)

We will turn off all output processing features by turning off the `OPOST`
flag. In practice, the `"\n"` to `"\r\n"` translation is likely the only output processing feature turned on by
default. `O` means it's an output flag, and I assume `POST` stands for "post-processing of output".

| **Commit Title** | **Location** |
|:-----------------|---------:|
| 13. Turn off all output processing | rawmode_unix.go|

```go

termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON | unix.ISIG | unix.IEXTEN)
termios.Iflag = termios.Iflag &^ (unix.IXON | unix.ICRNL)
//######## Lines to Add/Change ##########
termios.Oflag = termios.Oflag &^ (unix.OPOST)
//################################

```

## 14. Fix carriage returns

If you ran the program after turning off `OPOST`, you'll see that the newline characters we're printing are only moving
the cursor down, and not to the left side of the screen. To fix that, let's add carriage returns to our
`fmt.Printf()` statements.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| 14. Fix carriage returns | main.go|

```go 

//######## Lines to Add/Change ##########
func main() {

	origTermios, err := rawMode()
	if err != nil {
		fmt.Printf("Error: %s\r\n", err)
		os.Exit(1)
	}

	defer func() {
		if err := restore(origTermios); err != nil {
			fmt.Printf("Error: %s\r\n", err)
		}
	}()

	r := bufio.NewReader(os.Stdin)

	for {
		b, err := r.ReadByte()

		if err == io.EOF {
			break
		} else if err != nil {
			fmt.Printf("Error reading from Stdin: %s\r\n", err)
			os.Exit(1)
		}

		if isCntrl(b) {
			fmt.Printf("%d\r\n", b)
		} else {
			fmt.Printf("%d (%c)\r\n", b, b)
		}

		if b == 'q' {
			break
		}
	}
}
//################################

```

From now on, we'll have to write out the full `"\r\n"` whenever we want to start a new line.

## 15. Miscellaneous flags

Let's turn off a few more flags. `BRKINT`, `INPCK`, `ISTRIP`, and `CS8`.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| 15. Miscellaneous flags | main.go|

```go 

    termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON | unix.ISIG | unix.IEXTEN)

    //######## Lines to Add/Change ##########
    termios.Iflag = termios.Iflag &^ (unix.IXON | unix.ICRNL | unix.BRKINT | unix.INPCK | unix.ISTRIP)
    //################################

    termios.Oflag = termios.Oflag &^ (unix.OPOST)

    //######## Lines to Add/Change ##########
    termios.Cflag = termios.Cflag | unix.CS8
    //################################

```

This step probably won't have any observable effect for you, because these flags are either already turned off, or they
don't really apply to modern terminal emulators. But at one time or another, switching them off was considered (by
someone) to be part of enabling "raw mode", so we carry on the tradition (of whoever that someone was) in our program.

As far as I can tell:

* When `BRKINT` is turned on, a
  [break condition](https://www.cmrr.umn.edu/~strupp/serial.html#2_3_3) will cause a `SIGINT` signal to be sent to the
  program, like pressing `Ctrl-C`.
* `INPCK` enables parity checking, which doesn't seem to apply to modern terminal emulators.
* `ISTRIP` causes the 8th bit of each input byte to be stripped, meaning it will set it to `0`. This is probably already
  turned off.
* `CS8` is not a flag, it is a bit mask with multiple bits, which we set using the bitwise-OR (`|`) operator unlike all
  the flags we are turning off. It sets the character size (CS) to 8 bits per byte. On my system, it's already set that
  way.

## 16. Safe Exit

Now that we are able to get into raw mode and also restore safely, let's create a helper function that we can call
anywhere from our program to exit raw mode safely, show error messages if any and indicate to the OS whether we exited
cleanly or with errors.

To do this, we are going to take advantage of a feature of Go called
[First Class Functions](https://golang.org/doc/codewalk/functions/) in which we can treat functions as we would treat
any other variables and structs.

Let us first create a globally accessible variable `safeExit` which takes the type `func(error)`. In other words, we can
assign to this variable any function that takes an `error` as a parameter and returns nothing.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| 16. Safe Exit | main.go|

```go

import (
"bufio"
"fmt"
"io"
"os"
)

//######## Lines to Add/Change ##########
var safeExit func (error)
//################################

func isCntrl(b byte) bool {

```

Then after we get `origTermios` by calling `rawMode()`, we can set
`safeExit` to such an anonymous function that restores `origTermios`
and also prints an error message if `safeExit` was called with a non-`nil` error value similar to what we were doing
previously in the `defer` statement. One tweak here is we explictly now log error messages to `os.Stderr` which is where
by convention operating systems expect error messages to show up. To do this, we can use `fmt.Fprintf`
funciton which allows us to send output to anything that implements the standard `io.Writer` interface which `os.Stderr`
does. We take this opportunity to also make the same change in the code that handles failure to enter raw mode.

| **Commit Title** | **Location** |
|:-----------------|---------:|
| 16. Safe Exit | main.go|

```go

origTermios, err := rawMode()
if err != nil {

//######## Lines to Add/Change ##########
fmt.Fprintf(os.Stderr, "Error: %s\r\n", err)
//################################

os.Exit(1)
}

//######## Lines to Add/Change ##########
safeExit = func (err error) {
if errRestore := restore(origTermios); err != nil {
fmt.Fprintf(os.Stderr, "Error: disabling raw mode: %s\r\n", errRestore)
}

if err != nil {
fmt.Fprintf(os.Stderr, "Error: %s\r\n", err)
os.Exit(1)
}
os.Exit(0)
}
//################################

```

Finally, we can simply change the `defer` statement to call
`safeExit(nil)` to call it with no error and similarly change the places we need to exit with error to `safeExit(err)`

| **Commit Title** | **Location** |
|:-----------------|---------:|
| 16. Safe Exit | main.go|

```go

//######## Lines to Add/Change ##########
defer safeExit(nil)
//################################

r := bufio.NewReader(os.Stdin)

for {
b, err := r.ReadByte()

if err == io.EOF {
break
} else if err != nil {
//######## Lines to Add/Change ##########
safeExit(err)
//################################
}

if isCntrl(b) {
```

An easy way to make `rawMode()` fail is to give your program a text file or a pipe as the standard input instead of your
terminal. To give it a file as standard input, run `./gokilo < main.go`. To give it a pipe, run
`echo test | ./gokilo`. Both should result in an error like
`inappropriate ioctl for device`.

That just about concludes this chapter on entering raw mode. In the
[next chapter](/raw-input-and-output.html), we'll do some more low-level terminal input/output handling, and use that to
draw to the screen and allow the user to move the cursor around.
