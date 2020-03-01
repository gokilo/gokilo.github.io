---
layout: default
title: Raw Input and Output
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


