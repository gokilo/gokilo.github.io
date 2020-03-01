---
layout: default
title: Raw Input and Output
---

## Press <kbd>Ctrl-Q</kbd> to quit

Last chapter we saw that the <kbd>Ctrl</kbd> key combined with the alphabetic
keys seemed to map to bytes 1&ndash;26. We can use this to detect
<kbd>Ctrl</kbd> key combinations and map them to different operations in our
editor. We'll start by mapping <kbd>Ctrl-Q</kbd> to the quit operation.

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
