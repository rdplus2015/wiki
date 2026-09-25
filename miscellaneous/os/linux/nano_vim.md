# Linux System 

## Nano 

### Basic Commands

- **Open a file:** `nano filename`
- **Save changes:** `Ctrl + O`, then press `Enter`
- **Exit Nano:** `Ctrl + X`
- **Cut text:** `Ctrl + K`
- **Paste text:** `Ctrl + U`
- **Search for text:** `Ctrl + W`, then type search term and press `Enter`
- **Replace text:** `Ctrl + \`, then type search term, press `Enter`, then type replacement text, and press `Enter`
- **Go to a specific line:** `Ctrl + _`, then enter the line number and press `Enter`
- **Undo last action:** `Alt + U`
- **Redo last undone action:** `Alt + E`

### Navigation

- **Move cursor up:** `Ctrl + P`
- **Move cursor down:** `Ctrl + N`
- **Move cursor left:** `Ctrl + B`
- **Move cursor right:** `Ctrl + F`
- **Move to beginning of line:** `Ctrl + A`
- **Move to end of line:** `Ctrl + E`
- **Move to the top of the file:** `Ctrl + Y`
- **Move to the bottom of the file:** `Ctrl + V`

## Vim Cheatsheet

### Navigation

- **Move cursor left:** `h`
- **Move cursor down:** `j`
- **Move cursor up:** `k`
- **Move cursor right:** `l`

- **Move forward by one word:** `w`
- **Move backward by one word:** `b`

- **Move to the beginning of the line:** `^`
- **Move to the end of the line:** `$`

### Insertion and Deletion

- **Insert text before the cursor:** `i`
- **Insert text at the beginning of the line:** `I`
- **Insert text after the cursor:** `a`
- **Insert text at the end of the line:** `A`

- **Delete the character under the cursor:** `x`
- **Delete the word under the cursor:** `dw`
- **Delete (cut) the entire line:** `dd`
- **Delete from the cursor to the end of the line:** `D`

- **Copy the current line:** `yy`
- **Paste the contents of the clipboard:** `p`
- **Undo the last action:** `u`

### Main Commands

- **Save the file:** `:w`
- **Quit Vim:** `:q`
- **Save and quit:** `:wq` or `:x`
- **Show line numbers:** `:set nu`
- **Hide line numbers:** `:set nonu`
