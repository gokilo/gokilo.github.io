---
layout: default
title: Entering Raw Mode
---

# Entering Raw Mode

## 4. Get keypresses from user

Let's try and read keypresses from the user. 

| **Commit Title** | **File** |
|:-----------------|---------:|
| Get keypresses from user| main.go|

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

## 5. Press <kbd>q</kbd> to quit?

To demonstrate how canonical mode works, we'll have the program exit when it
reads a <kbd>q</kbd> keypress from the user. 

| **Commit Title** | **File** |
|:-----------------|---------:|
| Press q to quit | main.go|

```go

	for {

		//######## Lines to Add/Change ##########
		b, err := r.ReadByte()
		//#######################################

		if err == io.EOF {
			break
		} else if err != nil {
			fmt.Printf("Error reading from Stdin: %s\n", err)
			os.Exit(1)
		}

		//######## Lines to Add/Change ##########
		if b == 'q' {
			break
		}
		//#######################################
	}
}
```

To quit this program, you will have to type a line of text that includes a `q`
in it, and then press enter. The program will quickly read the line of text one
character at a time until it reads the `q`, at which point the `for` loop
will stop and the program will exit. Any characters after the `q` will be left
unread on the input queue, and you may see that input being fed into your shell
after your program exits.

## 6. Turn off echoing

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
of go called [build constraints](https://golang.org/pkg/go/build/#hdr-Build_Constraints). 

In code below, take a look at the very first line in the source code listing below which looks like this.
```go
// +build linux
```
This tells the Go compiler to use this file only for building Linux
targets. This way, we keep can make the Linux specific system calls
in functions in this file. Later when porting over GoKilo to
Windows, we can implement the same functions with Windows system
calls in a similar file guarded with `// +build windows` constraint. 

**Note**: In the listings below, we will be making changes in two
files in the same commit!

| **Commit Title** | **File** |
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
| Turn off echoing | main.go|

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

The `unix.ECHO` feature causes each key you type to be printed to the terminal, 
so you can see what you're typing. This is useful in canonical mode, but really
gets in the way when we are trying to carefully render a user interface in raw
mode. So we turn it off. This program does the same thing as the one in the
previous step, it just doesn't print what you are typing. You may be familiar
with this mode if you've ever had to type a password at the terminal, when
using `sudo` for example. You will still need to press `enter` after `q` to
quit.

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


## 7. Disable raw mode at exit

Let's be nice to the user and restore their terminal's original attributes when
our program exits. We'll save a copy of the `Termios` struct in its original
state, and use `unix.IoctlSetTermios()` to apply it to the terminal when 
the program exits with a new function called `restore()`.

We will make the changes to support this functionality in three steps
- Make `rawMode()` return the current terminal settings
- Create a `restore()` function to restore settings
- Adjust `main()` to enable/disable raw mode 


### 7a. Make `rawMode()` return the current terminal settings 

To make `rawMode()` return the current `Termios` settings
we will take advantage of Go's multpile return value feature to return
both `Termios` and the error value we are currently returning.

| **Commit Title** | **File** |
|:-----------------|---------:|
| Make rawMode return current terminal settings| rawmode_unix.go|


```go

// +build linux

package main

import (
	//######## Lines to Add/Change ##########
	"bytes"
	"encoding/gob"
	//#######################################
	"fmt"

	"golang.org/x/sys/unix"
)

//######## Lines to Add/Change ##########
func rawMode() ([]byte, error) {
//#######################################

	termios, err := unix.IoctlGetTermios(unix.Stdin, unix.TCGETS)
	if err != nil {
		//######## Lines to Add/Change ##########
		return nil, fmt.Errorf("could not fetch console settings: %s", err)
		//#######################################
	}

	//######## Lines to Add/Change ##########
	buf := bytes.Buffer{}
	if err := gob.NewEncoder(&buf).Encode(termios); err != nil {
		return nil, fmt.Errorf("could not serialize settings: %w", err)
	}
	//#######################################

	termios.Lflag = termios.Lflag &^ unix.ECHO

	if err := unix.IoctlSetTermios(unix.Stdin, unix.TCSETSF, termios); err != nil {
		//######## Lines to Add/Change ##########
		return nil, fmt.Errorf("could not set console settings: %s", err)
		//#######################################
	}

	//######## Lines to Add/Change ##########
	return buf.Bytes(), nil
	//#######################################
}


```

Since `Termios` is a Linux specific struct and is not defined
in Windows, we can make the function signature of `rawMode()` more 
platform independent by serializing the struct into more a generic 
type like byte slice. That way, when we port GoKilo to Windows, we
don't need ot make any changes in `main()` and merely implement a
Windows version of this function. In any case in our main program
we will not access this information beyond storing it and passing to
the platform specific `restore()` - so it can be a black box container.

To serialize, we will use Go's native `encoding/gob` package which 
serializes both the `Termios` data & information about the data
structure into a byte slice

**NOTE:**: At this stage, your code won't compile since our function
call in `main()` still uses the old function signature.

### 7b. Create a `restore()` function to restore settings

Having written the `rawMode()` function, `restore()` is fairly simple.
It uses a `gob.Decoder` to decode the previously serialized `Termios`
and use the `IoctlSetTermios()` system call to restore it. Add the function
to the end of the file.

| **Commit Title** | **File** |
|:-----------------|---------:|
| Create a restore function | rawmode_unix.go|

```go

//######## Lines to Add/Change ##########
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
//#######################################
```

### 7c. Adjust `main()` to enable/disable raw mode

In the `main()` function, we can now call the revised `rawMode()`
function and store the serialized original `Termios` settings in
a variable called `origTermios`. 

When the program exits, we can call `restore()` passing this 
variable to restore terminal settings. We do this by using the
`defer` keyword which registers this function to be called when
exiting the `main()` function.


| **Commit Title** | **File** |
|:-----------------|---------:|
| Adjust main to enable or disable raw mode| main.go|

```go

func main() {

    //######## Lines to Add/Change ##########
	origTermios, err := rawMode()
	if err != nil {
		fmt.Printf("Error: %s\n", err)
		os.Exit(1)
	}

	defer restore(origTermios)
    //################################

	r := bufio.NewReader(os.Stdin)

	// ---
}

```

## 8. Turn off canonical mode

There is an `ICANON` flag that allows us to turn off canonical mode. This means
we will finally be reading input byte-by-byte, instead of line-by-line. Now the program will quit as soon as you press <kbd>q</kbd>. 

| **Commit Title** | **File** |
|:-----------------|---------:|
| Turn off canonical mode| rawmode_unix.go|

```go

func rawMode() ([]byte, error) {

	// ---

    //######## Lines to Add/Change ##########
	termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON)
	//#######################################
	
	// ---
}

```


## 9. Display keypresses

To get a better idea of how input in raw mode works, let's print out each byte
that we read. We'll print each character's numeric ASCII value, as well as
the character it represents if it is a printable character.


### 9a. Detect Control Characters

Let's first create a small function `isCntrl()` to tests whether a character
is a control character. Control characters are nonprintable characters
that we don't want to print to the screen. ASCII codes 0&ndash;31 are 
all control characters, and 127 is also a control character. ASCII 
codes 32&ndash;126 are all printable. (Check out the
[ASCII table](http://asciitable.com) to see all of the characters.)

| **Commit Title** | **File** |
|:-----------------|---------:|
| Detect control characters | main.go|

```go
package main

import (
	// ---
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
	// ---
}

```

## 9b. Display Keypresses

`fmt.Printf()` can print multiple representations of a byte. `%d` tells it to
format the byte as a decimal number (its ASCII code), and `%c` tells it to
write out the byte directly, as a character.

| **Commit Title** | **File** |
|:-----------------|---------:|
| Display Keypresses| main.go|

```go

func main() {

	// ---

	for {

		b, err := r.ReadByte()

		if err == io.EOF {
			break
		} else if err != nil {
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
			break
		}
	}
}

```


This is a very useful program. It shows us how various keypresses translate
into the bytes we read. Most ordinary keys translate directly into the
characters they represent. But try seeing what happens when you press the arrow
keys, or <kbd>Escape</kbd>, or <kbd>Page Up</kbd>, or <kbd>Page Down</kbd>, or
<kbd>Home</kbd>, or <kbd>End</kbd>, or <kbd>Backspace</kbd>, or
<kbd>Delete</kbd>, or <kbd>Enter</kbd>. Try key combinations with
<kbd>Ctrl</kbd>, like <kbd>Ctrl-A</kbd>, <kbd>Ctrl-B</kbd>, etc.

You'll notice a few interesting things:

* Arrow keys, <kbd>Page Up</kbd>, <kbd>Page Down</kbd>, <kbd>Home</kbd>, and
  <kbd>End</kbd> all input 3 or 4 bytes to the terminal: `27`, `'['`, and then
  one or two other characters. This is known as an *escape sequence*. All
  escape sequences start with a `27` byte. Pressing <kbd>Escape</kbd> sends a
  single `27` byte as input.
* <kbd>Backspace</kbd> is byte `127`. <kbd>Delete</kbd> is a 4-byte escape
  sequence.
* <kbd>Enter</kbd> is byte `10`, which is a newline character, also known as
  `'\n'`.
* <kbd>Ctrl-A</kbd> is `1`, <kbd>Ctrl-B</kbd> is `2`, <kbd>Ctrl-C</kbd> is...
  oh, that terminates the program, right. But the <kbd>Ctrl</kbd> key
  combinations that do work seem to map the letters A&ndash;Z to the codes
  1&ndash;26.

On some consoles, if you happen to press <kbd>Ctrl-S</kbd>, you may find your program
seems to be frozen. What you've done is you've asked your program to [stop
sending you output](https://en.wikipedia.org/wiki/Software_flow_control). Press
<kbd>Ctrl-Q</kbd> to tell it to resume sending you output.

Also, if you press <kbd>Ctrl-Z</kbd> (or maybe <kbd>Ctrl-Y</kbd>), your program
will be suspended to the background. Run the `fg` command to bring it back to
the foreground but it may no longer be in raw mode. It may also quit immediately
after you do that, as a result of the underlying File reader returning error.

## 10. Turn off <kbd>Ctrl-C</kbd> and <kbd>Ctrl-Z</kbd> signals

By default, <kbd>Ctrl-C</kbd> sends a `SIGINT` signal to the current process
which causes it to terminate, and <kbd>Ctrl-Z</kbd> sends a `SIGTSTP` signal to
the current process which causes it to suspend. Let's turn off the sending of
both of these signals.

| **Commit Title** | **File** |
|:-----------------|---------:|
| Turn off Ctrl-C and Ctrl-Z signals| rawmode_unix.go|

```go

func rawMode() ([]byte, error) {

	// ---

	//######## Lines to Add/Change ##########
	termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON | unix.ISIG)
	//################################

	// ---
}

```

`ISIG` comes from `<termios.h>`. Like `ICANON`, it starts with `I` but isn't an
input flag.

Now <kbd>Ctrl-C</kbd> can be read as a `3` byte and <kbd>Ctrl-Z</kbd> can be
read as a `26` byte.

This also disables <kbd>Ctrl-Y</kbd> on macOS, which is like <kbd>Ctrl-Z</kbd>
except it waits for the program to read input before suspending it.


## 11. Disable <kbd>Ctrl-S</kbd> and <kbd>Ctrl-Q</kbd>

By default, <kbd>Ctrl-S</kbd> and <kbd>Ctrl-Q</kbd> are used for
[software flow control](https://en.wikipedia.org/wiki/Software_flow_control).
<kbd>Ctrl-S</kbd> stops data from being transmitted to the terminal until you
press <kbd>Ctrl-Q</kbd>. This originates in the days when you might want to
pause the transmission of data to let a device like a printer catch up. Let's
just turn off that feature.

| **Commit Title** | **File** |
|:-----------------|---------:|
| Disable Ctrl-S and Ctrl-Q | rawmode_unix.go|

```go

func rawMode() ([]byte, error) {

	// ---

    termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON | unix.ISIG)
    //######## Lines to Add/Change ##########
	termios.Iflag = termios.Iflag &^ (unix.IXON)
	//################################
	
	// ---
}

```

The `I` in `IXON` stands for "input flag" (which it is, unlike the 
other `L` flags we've seen so far) and `XON` comes from the names of
the two control characters that <kbd>Ctrl-S</kbd> and <kbd>Ctrl-Q</kbd>
produce: `XOFF` to pause transmission and `XON` to resume transmission.

Now <kbd>Ctrl-S</kbd> can be read as a `19` byte and <kbd>Ctrl-Q</kbd> can be
read as a `17` byte.


## 12. Disable <kbd>Ctrl-V</kbd>

On some systems, when you type <kbd>Ctrl-V</kbd>, the terminal waits for you to
type another character and then sends that character literally. For example,
before we disabled <kbd>Ctrl-C</kbd>, you might've been able to type
<kbd>Ctrl-V</kbd> and then <kbd>Ctrl-C</kbd> to input a `3` byte. We can turn
off this feature using the `IEXTEN` flag.

Turning off `IEXTEN` also fixes <kbd>Ctrl-O</kbd> in macOS, whose terminal
driver is otherwise set to discard that control character. It is another
flag that starts with `I` but actually belongs in the `Lflag` field.

| **Commit Title** | **File** |
|:-----------------|---------:|
| Disable Ctrl-V | rawmode_unix.go|

```go

func rawMode() ([]byte, error) {

	// ---

    //######## Lines to Add/Change ##########
	termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON | unix.ISIG | unix.IEXTEN)
	//################################
	termios.Iflag = termios.Iflag &^ (unix.IXON)
	
	// ---
}

```

<kbd>Ctrl-V</kbd> can now be read as a `22` byte, and <kbd>Ctrl-O</kbd> as a
`15` byte.


## 13. Fix <kbd>Ctrl-M</kbd>

If you run the program now and go through the whole alphabet while holding down
<kbd>Ctrl</kbd>, you should see that we have every letter except <kbd>M</kbd>.
<kbd>Ctrl-M</kbd> is weird: it's being read as `10`, when we expect it to be
read as `13`, since it is the 13th letter of the alphabet, and
<kbd>Ctrl-J</kbd> already produces a `10`. What else produces `10`? The
<kbd>Enter</kbd> key does.

It turns out that the terminal is helpfully translating any carriage returns
(`13`, `'\r'`) inputted by the user into newlines (`10`, `'\n'`). Let's turn
off this feature.

| **Commit Title** | **File** |
|:-----------------|---------:|
| Fix Ctrl-M | rawmode_unix.go|

```go

func rawMode() ([]byte, error) {

	// ---


	termios.Lflag = termios.Lflag &^ (unix.ECHO | unix.ICANON | unix.ISIG | unix.IEXTEN)
    //######## Lines to Add/Change ##########
	termios.Iflag = termios.Iflag &^ (unix.IXON | unix.ICRNL)
	//################################
	
	// ---
}

```

The `I` in `ICRNL` stands for "input flag", `CR` stands
for "carriage return", and `NL` stands for "new line".

Now <kbd>Ctrl-M</kbd> is read as a `13` (carriage return), and the
<kbd>Enter</kbd> key is also read as a `13`.

## 13. Turn off all output processing

It turns out that the terminal does a similar translation on the output side.
It translates each newline (`"\n"`) we print into a carriage return followed by
a newline (`"\r\n"`). The terminal requires both of these characters in order
to start a new line of text. The carriage return moves the cursor back to the
beginning of the current line, and the newline moves the cursor down a line,
scrolling the screen if necessary. (These two distinct operations originated in
the days of typewriters and
[teletypes](https://en.wikipedia.org/wiki/Teleprinter).)

We will turn off all output processing features by turning off the `OPOST`
flag. In practice, the `"\n"` to `"\r\n"` translation is likely the only output
processing feature turned on by default. `O` means it's an output flag,
and I assume `POST` stands for "post-processing of output".

| **Commit Title** | **File** |
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

If you ran the program after turning off `OPOST`, you'll see that the newline
characters we're printing are only moving the cursor down, and not to the left
side of the screen. To fix that, let's add carriage returns to our 
`fmt.Printf()` statements.

| **Commit Title** | **File** |
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

From now on, we'll have to write out the full `"\r\n"` whenever we want
to start a new line.


## 15. Miscellaneous flags

Let's turn off a few more flags. `BRKINT`, `INPCK`, `ISTRIP`, and `CS8`.

| **Commit Title** | **File** |
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

This step probably won't have any observable effect for you, because these
flags are either already turned off, or they don't really apply to modern
terminal emulators. But at one time or another, switching them off was
considered (by someone) to be part of enabling "raw mode", so we carry on the
tradition (of whoever that someone was) in our program.

As far as I can tell:

* When `BRKINT` is turned on, a
  [break condition](https://www.cmrr.umn.edu/~strupp/serial.html#2_3_3) will
  cause a `SIGINT` signal to be sent to the program, like pressing `Ctrl-C`.
* `INPCK` enables parity checking, which doesn't seem to apply to modern
  terminal emulators.
* `ISTRIP` causes the 8th bit of each input byte to be stripped, meaning it
  will set it to `0`. This is probably already turned off.
* `CS8` is not a flag, it is a bit mask with multiple bits, which we set using
  the bitwise-OR (`|`) operator unlike all the flags we are turning off. It
  sets the character size (CS) to 8 bits per byte. On my system, it's already
  set that way.

## 16. Safe Exit

Now that we are able to get into raw mode and also restore safely, 
let's create a helper function that we can call anywhere from our program
to exit raw mode safely, show error messages if any and indicate to the
OS whether we exited cleanly or with errors. 

To do this, we are going to take advantage of a feature of Go called
[First Class Functions](https://golang.org/doc/codewalk/functions/) in which
we can treat functions as we would treat any other variables and structs.

Let us first create a globally accessible variable `safeExit` which takes
the type `func(error)`. In other words, we can assign to this variable any
function that takes an `error` as a parameter and returns nothing.

| **Commit Title** | **File** |
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
var safeExit func(error)
//################################

func isCntrl(b byte) bool {

```

Then after we get `origTermios` by calling `rawMode()`, we can set
`safeExit` to such an anonymous function that restores `origTermios` 
and also prints an error message if `safeExit` was called with a 
non-`nil` error value similar to what we were doing previously in
the `defer` statement. One tweak here is we explictly now log error
messages to `os.Stderr` which is where by convention operating systems
expect error messages to show up. To do this, we can use `fmt.Fprintf`
funciton which allows us to send output to anything that implements
the standard `io.Writer` interface which `os.Stderr` does. We take this
opportunity to also make the same change in the code that handles
failure to enter raw mode.

| **Commit Title** | **File** |
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
	safeExit = func(err error) {
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
`safeExit(nil)` to call it with no error and similarly change
the places we need to exit with error to `safeExit(err)`

| **Commit Title** | **File** |
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

An easy way to make `rawMode()` fail is to give your program a text file or a
pipe as the standard input instead of your terminal. To give it a file as
standard input, run `./gokilo < main.go`. To give it a pipe, run
`echo test | ./gokilo`. Both should result in an error like 
`inappropriate ioctl for device`.

That just about concludes this chapter on entering raw mode. In the
[next chapter](/raw-input-and-output.html), we'll do some more low-level
terminal input/output handling, and use that to draw to the screen and allow
the user to move the cursor around.
