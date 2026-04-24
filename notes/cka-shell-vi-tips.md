# CKA Notes: Shell and Vi Tips

Shell speed matters during Kubernetes labs. These are the shell and `vi` shortcuts that came up repeatedly while working through exercises.

## Shell History Reuse

- Repeat the last command: `!!`
- Reuse the last command and pipe it: `!! | grep hello`
- Preview history expansion without running it: `history -p !!`
- Print the last command from history: `fc -ln -1`
- Print the second last command: `fc -ln -2`
- Print a range of recent commands: `fc -ln -5 -1`

Write the previous command into a script file:

```bash
fc -ln -1 | sed 's/^[[:space:]]*//' > answer.sh
```

Notes:

- `fc -l` lists history
- `fc -n` removes history numbers
- negative indexes count backward from the current command
- `fc -ln` pads output, so strip leading spaces before writing to a file

## Shell Line Editing

These are useful when a long `kubectl` command is wrong and you want to fix it fast.

- `Ctrl-u`: delete from cursor to start of line
- `Ctrl-k`: delete from cursor to end of line
- `Ctrl-w`: delete the previous word
- `Ctrl-a`: move to start of line
- `Ctrl-e`: move to end of line

Quick pattern:

```text
Ctrl-a, then Ctrl-k
```

This clears the whole prompt line.

## echo and tee

- `echo` prints text to standard output
- `tee` reads from standard input and writes to both standard output and a file

Examples:

```bash
echo "hello"
echo "hello" | tee out.txt
echo "hello" | tee -a out.txt
```

This is useful when you want visible proof in logs and a file written at the same time.

## vi Movement

- `h j k l`: left, down, up, right
- `w`: next word start
- `b`: previous word start
- `e`: end of word
- `0`: start of line
- `^`: first non-blank character
- `$`: end of line
- `gg`: top of file
- `G`: bottom of file
- `H`: top of screen
- `M`: middle of screen
- `L`: bottom of screen
- `f{char}`: find character forward
- `;`: repeat last find
- `Ctrl-f`: page down
- `Ctrl-b`: page up
- `Ctrl-d`: half page down
- `Ctrl-u`: half page up

## vi Editing

- `Esc`: return to normal mode
- `i`: insert before cursor
- `a`: insert after cursor
- `o`: open new line below
- `O`: open new line above
- `x`: delete character under cursor
- `dd`: delete or cut current line
- `Ndd`: delete or cut `N` lines
- `yy`: yank current line
- `Nyy`: yank `N` lines
- `p`: paste below or after
- `P`: paste above or before
- `u`: undo
- `Ctrl-r`: redo
- `.`: repeat the last change

Important note:

- `dd` deletes and also stores the text in the unnamed register, so it behaves like cut

Delete without overwriting your last yank:

```vim
"_dd
```

## vi Save and Quit

- `:w`: save
- `:q`: quit
- `:wq`: save and quit
- `:q!`: quit without saving

## vi YAML Paste Recovery

Recommended `~/.vimrc` line:

```vim
set ts=2 sw=2 et
```

Paste without auto-indent mangling:

```vim
:set paste
" paste your YAML
:set nopaste
```

Recover from tabs or hidden characters in YAML:

```vim
:set list
:set expandtab
:retab
:set nolist
```

Notes:

- `:set list` shows hidden characters like `^I` for tabs
- `:set expandtab` makes typed tabs become spaces
- `:retab` converts existing tabs to spaces
- `:set nolist` hides the markers again

## vi Block Indent

Visual block shifting:

```vim
V
j
>
```

Or:

```vim
V
k
<
```

Direct line-count shifting without visual selection:

```vim
5>>
5<<
```

Notes:

- `V` starts visual line selection
- `j` / `k` expands the selected block
- `>` indents the selected block one shiftwidth
- `<` unindents the selected block one shiftwidth
- `5>>` indents the current line plus the next 4 lines
- `5<<` unindents the current line plus the next 4 lines
- `5>>` / `5<<` is faster when you already know roughly how many lines must shift

## vi Delete Patterns

- `dw`: delete forward by word
- `dW`: delete forward by WORD
- `db`: delete backward by word
- `dB`: delete backward by WORD
- `de`: delete to end of word
- `d0`: delete from cursor to start of line
- `d^`: delete from cursor to first non-blank character
- `D`: delete from cursor to end of line
- `dt{char}`: delete until before a character
- `df{char}`: delete through a character

Example:

```vim
dtw
```

This deletes from the cursor until before `w`. It is useful for removing spaces or other characters up to a target character on the same line.

## vi Search and Replace

Search:

- `/text`: search forward
- `?text`: search backward
- `n`: next match
- `N`: previous match

Replace in the whole file:

```vim
:%s/old/new/g
```

Replace with confirmation:

```vim
:%s/old/new/gc
```

## Kubernetes Exam Habits

- Use short aliases such as `k` only if they already exist in the environment
- Reuse recent commands with `!!` and `fc` instead of retyping long `kubectl` pipelines
- Use `tee` when you want both terminal proof and a file written inside a container command
- Keep `vi` deletes and movement efficient so manifest edits stay fast under time pressure
