---
layout: default
title: Raw Input and Output
nav_order: 4
---

## 17. Refactor keyboard input

Let's make a function for low-level keypress reading, and another function for
mapping keypresses to editor operations moving them out of the main loop.

We'll create `readKey()` function in a new file `terminal.go` which for now
simply mirrors what we're doing in `main()`. The only tweak is that we'll
initialize the `bufio.Reader` just once as a global variable `keyReader`. We'll
expand this `ReadKey()` function to handle escape sequences, which involves
reading multiple runes that represent a single keypress, as is the case with the
arrow keys.

| **Commit Title** | **File** |
|:-----------------|---------:|
| Refactor keyboard input | terminal.go|

```diff
+package main
+
+import (
+	"bufio"
+	"os"
+)
+
+var keyReader = bufio.NewReader(os.Stdin)
+
+func readKey() (rune, error) {
+	ru, _, err := keyReader.ReadRune()
+	return ru, err
+}

```
We will create another file `input.go` and define `processKey()` that to handle
key presses. Again we've simply moved existing functionality from `main()` here
though note that we've dropped the EOF character handling as we're alreary in
raw mode. Later, we will map various key combinations and other special keys
to different editor functions, and insert any alphanumeric and other printable
keys' characters into the text that is being edited.

Note that `readKey()` belongs in `terminal.go` because it deals with 
low-level terminal input, whereas `processKey()` belongs in 
`input.go` because it deals with mapping keys to editor functions
at a much higher level.

| **Commit Title** | **File** |
|:-----------------|---------:|
| Refactor keyboard input | input.go|

```diff
+package main
+
+import (
+	"fmt"
+	"unicode"
+)
+
+func processKey(key rune) {
+
+	if unicode.IsControl(key) {
+		fmt.Printf("%d\r\n", key)
+	} else {
+		fmt.Printf("%d (%c)\r\n", key, key)
+	}
+
+	if key == 'q' {
+		safeExit(nil)
+	}
+}
```

Finally, we'll massively simplify `main()`, and we will try to keep it that way.

| **Commit Title** | **File** |
|:-----------------|---------:|
| Refactor keyboard input | main.go|

```diff
 package main
 
 import (
-	"bufio"
 	"fmt"
-	"io"
 	"os"
-	"unicode"
 )
 
 var globalState = struct {
 	restoreTerminal func()
 }{
 	nil,
 }
 
 func safeExit(err error) {
 	if globalState.restoreTerminal != nil {
 		globalState.restoreTerminal()
 	}
 
 	if err != nil {
 		fmt.Fprintf(os.Stderr, "Error: %s\r\n", err)
 		os.Exit(1)
 	}
 	os.Exit(0)
 }
 
 func main() {
 
 	restoreFunc, err := rawMode()
 	if err != nil {
 		safeExit(err)
 	}
 	globalState.restoreTerminal = restoreFunc
 	defer safeExit(nil)
 
-	r := bufio.NewReader(os.Stdin)
-
 	for {
-		ru, _, err := r.ReadRune()
-
-		if err == io.EOF {
-			safeExit(nil)
-		} else if err != nil {
+		key, err := readKey()
+		if err != nil {
 			safeExit(err)
 		}
-		if unicode.IsControl(ru) {
-			fmt.Printf("%d\r\n", ru)
-		} else {
-			fmt.Printf("%d (%c)\r\n", ru, ru)
-		}
-
-		if ru == 'q' {
-			safeExit(nil)
-		}
+		processKey(key)
 	}
 }

```

## 18. Press <kbd>Ctrl-Q</kbd> to quit

Last chapter we saw that the <kbd>Ctrl</kbd> key combined with the alphabetic
keys seemed to map to bytes 1&ndash;26. We can use this to detect
<kbd>Ctrl</kbd> key combinations and map them to different operations in our
editor. We'll start by mapping <kbd>Ctrl-Q</kbd> to the quit operation and we'll
start re-structuring the keyboard input functions to handle more complex cases.

| **Commit Title** | **File** |
|:-----------------|---------:|
| Press Ctrl-Q to quit | terminal.go|

```diff
 package main
 
 import (
 	"bufio"
 	"os"
+	"unicode"
+)
+
+type KeyType int
+
+const (
+	RegKey = iota
+	CtrlKey
+)
+
+type Key struct {
+	KeyType KeyType
+	Key     rune
+}
+
+const (
+	KeyCtrlQ = 17
 )
 
 var keyReader = bufio.NewReader(os.Stdin)
 
-func readKey() (rune, error) {
+func readKey() (Key, error) {
 	ru, _, err := keyReader.ReadRune()
-	return ru, err
+	if err != nil {
+		return Key{}, err
+	}
+
+	switch {
+	case unicode.IsControl(ru):
+		return Key{CtrlKey, ru}, err
+	default:
+		return Key{RegKey, ru}, err
+
+	}
 }

```

There's a lot going on here, so let's break it down piece by piece. First we're
defining `KeyType` as a new data type based on `int` and then declaring
constants of this type for regular and control characters. We could just use
`int` but declaring a new type helps with clarity & helps IDEs.

We then create a struct `Key` which will represent the pressed key instead of a
plain rune to allow us to treat special keys the same way. It will have a field
`KeyType` to describe the type of the key and a `rune` which holds the actual key
value. Since runes are [simply 32 bit integers](https://blog.golang.org/strings#TOC_5.)
we'll also use they same field to hold values representing special keys and
we're defining the constant `KeyCtrlQ` as `17` as it's the 17th letter in the
English alphabet and we've already seen <kbd>Ctrl<kbd> characters map to codes 1
to 26. 

Finally, we edit the `readKey()` function to return a `Key` instead of a plain rune.


We then need to change `processKey()` to work on the new Key type rather than a
`rune`. The changes here are much simpler, since now we're getting the key type
directly.

| **Commit Title** | **File** |
|:-----------------|---------:|
| Press Ctrl-Q to quit | input.go|

```diff
 package main
 
 import (
 	"fmt"
-	"unicode"
 )
 
-func processKey(key rune) {
+func processKey(key Key) {
 
-	if unicode.IsControl(key) {
-		fmt.Printf("%d\r\n", key)
-	} else {
-		fmt.Printf("%d (%c)\r\n", key, key)
+	switch key.KeyType {
+	case CtrlKey:
+		fmt.Printf("Control Key: %d\r\n", key.Key)
+	case RegKey:
+		fmt.Printf("Regular Key: %d (%c)\r\n", key.Key, key.Key)
 	}
 
-	if key == 'q' {
+	if key.KeyType == CtrlKey && key.Key == KeyCtrlQ {
 		safeExit(nil)
 	}
 }
 ```

Interstingly enough, there are no changes to be made in `main.go`. Since we're
using the `:=` short assignment statement to capture returned values from
`readKey()`, the compiler automatically infers its type correctly.

```go
		key, err := readKey()
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
