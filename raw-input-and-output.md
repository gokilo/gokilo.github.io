---
layout: default
title: Raw Input and Output
nav_order: 4
---

## 17. Press <kbd>Ctrl-Q</kbd> to quit

Last chapter we saw that the <kbd>Ctrl</kbd> key combined with the alphabetic
keys seemed to map to bytes 1&ndash;26. We can use this to detect
<kbd>Ctrl</kbd> key combinations and map them to different operations in our
editor. We'll start by mapping <kbd>Ctrl-Q</kbd> to the quit operation.
We'll also stop printing out keypresses at this point.


| **Commit Title** | **File** |
|:-----------------|---------:|
| 17. Press Ctrl-Q to quit | main.go|

```go

func isCntrl(b byte) bool {
	if b <= 0x1f || b == 0x7f {
		return true
	}
	return false
}

//######## Lines to Add/Change ##########
func cntrlKey(b byte) byte {
	return b & 0x1f
}
//#######################################

func main() {

```

| **Commit Title** | **File** |
|:-----------------|---------:|
| 17. Press Ctrl-Q to quit | main.go|

```go


	for {
		b, err := r.ReadByte()

		if err == io.EOF {
			break
		} else if err != nil {
			safeExit(err)
		}

        //######## Lines to Add/Change ##########
		if b == cntrlKey('q') {
			break
        }
        //#######################################
    }

```

The `ctrlKey()` function bitwise-ANDs a character with the value `00011111`, in
binary. We generally specify bitmasks using hexadecimal, for conciseness and
readiability. In other words, it sets the upper 3 bits of the character
to `0`. This mirrors what the <kbd>Ctrl</kbd> key does in the terminal: 
it strips bits 5 and 6 from whatever key you press in combination with
<kbd>Ctrl</kbd>, and sends that. (By convention, bit numbering starts from 0.) 
The ASCII character set seems to be designed this way on purpose. 
(It is also similarly designed so that you can set and clear bit 5 to switch 
between lowercase and uppercase.)

## 18. Refactor keyboard input

Let's make a function for low-level keypress reading, and another function for
mapping keypresses to editor operations. 

We'll create the `ReadKey()` routine in a new file `terminal.go`.
To read user key presses one byte at a time, we've been using a 
`bufio.Reader` instantiated inside `main()`. To generalize, we'll
create a `KeyReader` type embedding `bufio.Reader` that's initialized
to read from terminal when `NewKeyReader()` is called. Finally we'll
define the `ReadKey()` function under this type. This way, we
avoid global variables.

We'll expand this `ReadKey()` function to handle escape sequences,
which involves reading multiple bytes that represent a single keypress,
as is the case with the arrow keys.

| **Commit Title** | **File** |
|:-----------------|---------:|
| 17. Refactor keyboard input | terminal.go|

```go

package main

import (
	"bufio"
	"os"
)

// KeyReader reads key-presses fron terminal
type KeyReader struct {
	*bufio.Reader
}

// NewKeyReader initializes a key reader and sets it
// to read key presses from os.Stdin
func NewKeyReader() *KeyReader {
	return &KeyReader{
		bufio.NewReader(os.Stdin),
	}
}

// ReadKey reads a single key from the keyboard
func (kr *KeyReader) ReadKey() (byte, error) {
	return kr.ReadByte()
}

```

We will create another file `input.go` and define  `processKeyPress()` that
waits for a keypress from a `KeyReader`, and then handles it. Later, it
will map various <kbd>Ctrl</kbd> key combinations and other special keys to
different editor functions, and insert any alphanumeric and other printable
keys' characters into the text that is being edited.

Note that `ReadKey()` belongs in `terminal.go` because it deals with 
low-level terminal input, whereas `processKeypress()` belongs in 
`input.go` because it deals with mapping keys to editor functions
at a much higher level.

| **Commit Title** | **File** |
|:-----------------|---------:|
| 17. Refactor keyboard input | input.go|

```go

package main

import (
	"io"
)

func processKeyPress(kr *KeyReader) {

	b, err := kr.ReadKey()

	if err == io.EOF {
		safeExit(nil)
	} else if err != nil {
		safeExit(err)
	}

	if b == cntrlKey('q') {
		safeExit(nil)
	}
}

```

Now we have vastly simplified `main()`, and we will try to keep it that way.
Note that you will also want to remove imports of `bufio` and `io` which
would have been required previously to compile successfully since 

| **Commit Title** | **File** |
|:-----------------|---------:|
| 17. Refactor keyboard input | input.go|


```go

import (
	// "bufio"  <- REMOVE
	"fmt"
	// "io"     <- REMOVE
	"os"
)

// ---

func main(){

    // ---

    //######## Lines to Add/Change ##########
    kr := NewKeyReader()

    for {
        processKeyPress(kr)
    }
    //######## Lines to Add/Change ##########
}
```

## 18. Clear the screen

We're going to render the editor's user interface to the screen after each
keypress. Let's start by just clearing the screen.


| **Commit Title** | **File** |
|:-----------------|---------:|
| 18. Clear the screen | screen.go|

```go

package main 

import (
	"fmt"
	"os"
)

func refreshScreen(){
	fmt.Fprint(os.Stdout, "\x1b[2J")
}

```

| **Commit Title** | **File** |
|:-----------------|---------:|
| 18. Clear the screen | main.go|

```go

// --

func main() {

	// ---

	kr := NewKeyReader()

	for {

		//######## Lines to Add/Change ##########
		refreshScreen()
		//####################################### 

		processKeyPress(kr)
	}
}

```

We're using `fmt.Fprint()` function from the `fmt` package to
send characters out to the screen via `os.Stdout`. We are sending
for bytes to the screen. The first byte is `\x1b`, which is the
escape character, or `27` in decimal. (Try and remember `\x1b`, 
we will be using it a lot.) The other three bytes are `[2J`.

We are writing an *escape sequence* to the terminal. Escape sequences always
start with an escape character (`27`) followed by a `[` character. Escape
sequences instruct the terminal to do various text formatting tasks, such as
coloring text, moving the cursor around, and clearing parts of the screen.

We are using the `J` command
([Erase In Display](http://vt100.net/docs/vt100-ug/chapter3.html#ED)) to clear
the screen. Escape sequence commands take arguments, which come before the
command. In this case the argument is `2`, which says to clear the entire
screen. `<esc>[1J` would clear the screen up to where the cursor is, and
`<esc>[0J` would clear the screen from the cursor up to the end of the screen.
Also, `0` is the default argument for `J`, so just `<esc>[J` by itself would
also clear the screen from the cursor to the end.

For our text editor, we will be mostly using
[VT100](https://en.wikipedia.org/wiki/VT100) escape sequences, which
are supported very widely by modern terminal emulators. See the
[VT100 User Guide](http://vt100.net/docs/vt100-ug/chapter3.html) for complete
documentation of each escape sequence.

If we wanted to support the maximum number of terminals out there, we could use
the [ncurses](https://en.wikipedia.org/wiki/Ncurses) library, which uses the
[terminfo](https://en.wikipedia.org/wiki/Terminfo) database to figure out the
capabilities of a terminal and what escape sequences to use for that particular
terminal.


## 19. Reposition the cursor

You may notice that the `<esc>[2J` command left the cursor at the bottom of the
screen. Let's reposition it at the top-left corner so that we're ready to draw
the editor interface from top to bottom. We'll also start commenting these on
these escape sequences for easy future reference.

| **Commit Title** | **File** |
|:-----------------|---------:|
| 19. Reposition the cursor | screen.go|

```go
// --
func refreshScreen(){

	//######## Lines to Add/Change ##########
	// Clear the Screen
	fmt.Fprint(os.Stdout, "\x1b[2J")

	// Reposition the cursor
	fmt.Fprint(os.Stdout, "\x1b[H")
	//####################################### 
}
```

This escape sequence is only `3` bytes long, and uses the `H` command
([Cursor Position](http://vt100.net/docs/vt100-ug/chapter3.html#CUP)) to
position the cursor. The `H` command actually takes two arguments: the row
number and the column number at which to position the cursor. So if you have an
80&times;24 size terminal and you want the cursor in the center of the screen, you could
use the command `<esc>[12;40H`. (Multiple arguments are separated by a `;`
character.) The default arguments for `H` both happen to be `1`, so we can
leave both arguments out and it will position the cursor at the first row and
first column, as if we had sent the `<esc>[1;1H` command. (Rows and columns are
numbered starting at `1`, not `0`.)

## 20. Clear the screen on exit

Let's clear the screen and reposition the cursor when our program exits. If an
error occurs in the middle of rendering the screen, we don't want a bunch of
garbage left over on the screen, and we don't want the error to be printed
wherever the cursor happens to be at that point.

| **Commit Title** | **File** |
|:-----------------|---------:|
| 20. Clear the screen on exit| main.go|

```go
// --
func main(){
	// --
	safeExit = func(err error) {

		//######## Lines to Add/Change ##########
		fmt.Fprint(os.Stdout, "\x1b[2J")
		fmt.Fprint(os.Stdout , "\x1b[H")
		//####################################### 

		if errRestore := restore(origTermios); err != nil {
			fmt.Fprintf(os.Stderr, "Error: disabling raw mode: %s\r\n", errRestore)
		}
		// --
	}
	// --
}

```

We exit only by the `safeExit()` function, so we have to have to add the commands
to clear screen and reposition cursor only there. 


## 21. Tildes

It's time to start drawing. Let's draw a column of tildes (`~`) on the left
hand side of the screen, like [vim](http://www.vim.org/) does. In our text
editor, we'll draw a tilde at the beginning of any lines that come after the
end of the file being edited.
| **Commit Title** | **File** |
|:-----------------|---------:|
| 21. Tildes | screen.go|

```go

import (
	//--
)

//######## Lines to Add/Change ##########
func drawRows(){
	for y := 0; y < 24; y++{
		fmt.Fprint(os.Stdout, "~\r\n")
	}
}
//####################################### 

func refreshScreen(){

	// Clear the Screen
	fmt.Fprint(os.Stdout, "\x1b[2J")

	// Reposition the cursor
	fmt.Fprint(os.Stdout, "\x1b[H")

	//######## Lines to Add/Change ##########
	drawRows()

	// Reposition the cursor
	fmt.Fprint(os.Stdout, "\x1b[H")
	//####################################### 
}
```

`drawRows()` will handle drawing each row of the buffer of text being
edited. For now it draws a tilde in each row, which means that row is not part
of the file and cannot contain any text.

We don't know the size of the terminal yet, so we don't know how many rows to
draw. For now we just draw `24` rows.

After we're done drawing, we do another `<esc>[H` escape sequence to reposition
the cursor back up at the top-left corner.

## 22. State variables

Our next goal is to get the size of the terminal, so we know how many rows to
draw in `drawRows()`. These kinds of variables represent the **state** of the 
running instance of the program. Storing state like this in global variables
isn't great practice but luckily Go being a fairly modern language has the
concepts of structs and methods. So let's create a `Screen` struct that will
store stateful data like the number or rows and columns in it and a `NewScreen()`
function to initialize a screen. 

| **Commit Title** | **File** |
|:-----------------|---------:|
| 22. State Variables | screen.go|

```go
package main 

import (
	"fmt"
	"os"
)

type Screen struct{
	Rows int
	Cols int
}

func NewScreen(rows, cols int) *Screen{
	return &Screen{rows, cols}
}

func (s *Screen) DrawRows(){
	for y := 0; y < s.Rows; y++{
		fmt.Fprint(os.Stdout, "~\r\n")
	}
}

func (s *Screen) Refresh(){

	// Clear the Screen
	fmt.Fprint(os.Stdout, "\x1b[2J")

	// Reposition the cursor
	fmt.Fprint(os.Stdout, "\x1b[H")

	s.DrawRows()

	// Reposition the cursor
	fmt.Fprint(os.Stdout, "\x1b[H")
}
```

We can make functions like `DrawRows()` and `Refresh()` that use these
screen state variables methods of the `Screen` struct. While Go is not
an object oriented language, this gives us nice idioms like 
`Screen.Refresh()` or `Screen.DrawRows()`.

Note the capitalization of the first Letters of the struct & the functions
indicating **exported** symbols. While in this case it doesn't matter
much as all private & public methods are accessible within the same 
package, creating methods this way lets us potentially modularize
these in the future. Also note the use of Pointer method receivers
to avoid making copies of the underlying struct when calling.

Finally, let's initialize a `Screen` struct variable in our `main()`
and the call its `Refresh()` method. For now let's hard code values
of 24 rows and 80 cols. In the next section, we'll figure out the 
actual screen size.

| **Commit Title** | **File** |
|:-----------------|---------:|
| 22. State Variables | screen.go|

```go
// --

func main(){

	//--
	
	kr := NewKeyReader()

	//######## Lines to Add/Change ##########
	screen := NewScreen(24, 80)
	//####################################### 

	for {
		//######## Lines to Add/Change ##########
		screen.Refresh()
		//####################################### 

		processKeyPress(kr)
	}
}

```
