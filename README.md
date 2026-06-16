# Dunnet-like Text Adventure Game

> [English] | [繁體中文](README_zh-TW.md)

**Main program file**: `dunnet`

**Programming language**: C Shell (`csh` / `tcsh`)

**Operating system**: Ubuntu 20.04.6 LTS

**Linux kernel version**: 5.15.0-139-generic

## 1. Project Overview

This project is an assignment for the UNIX system programming course. The assignment requires implementing a text adventure game similar to **Dunnet**. All operations and output results should be consistent with the original Dunnet. However, the full game flow does not need to be completed. The required implementation only needs to allow the player to use pokey to find the hidden treasure in the forest.

This project is written as a `csh` script. Players can move between different rooms, inspect objects, pick up or drop items, and progress through the game by interacting with objects in the scene. The game also includes a computer named pokey that simulates a UNIX environment. After starting pokey and logging in successfully, the player can enter simplified UNIX commands such as `ls`, `cd`, `pwd`, `cat`, `uncompress`, and so on.

---

## 2. How to Run

### Runtime Environment

A Linux or UNIX system is required, and the following must be supported:

- `csh` or `tcsh`
- `tar`
- `gunzip`
- Common UNIX utilities, such as `sed`, `tr`, `grep`, `ls`, `pwd`, `mv`, `touch`, and `chmod`

### Execution Steps

Place `dunnet` and `filesystem.tar` in a same directory.

Then enter the following in the terminal:

```bash
chmod +x dunnet

./dunnet
```

This will start the game.

### Testing Steps

First, make sure that `emacs` has been installed on the current Linux or UNIX system:

```bash
sudo apt update && sudo apt install -y emacs
```

Open a terminal and enter:

```bash
emacs -batch -l dunnet
```

This will start the original Dunnet, you can check the output is the same as the self-made Dunnet or not.

#### Checking the Output of the Custom Dunnet Implementation by Entering Commands Files

To check whether the custom Dunnet output is consistent with the original Dunnet, follow the steps below, or refer to the ReadMe.md file in the demo files directory.

First, place `dunnet`, `filesystem.tar`, and the three test files in the same directory.

Open a terminal and enter:

```bash
./dunnet < testactions > my_output.txt
```

This writes the test result to `my_output.txt`. The file name `testactions` can be replaced with another test file name.

Then copy all test files to another directory.

Open a terminal and enter:

```bash
emacs -batch -l dunnet < testactions > standard_output.txt
```

This writes the execution result of the original Dunnet to `standard_output.txt`.

Next, move `my_output.txt` and `standard_output.txt` into a same directory. Open a terminal and enter:

```bash
diff -y -w my_output.txt standard_output.txt
```

This allows you to view the differences between the output of the custom Dunnet implementation and the output of the original Dunnet.

### Program Initialization Flow

When the game starts, the program first changes the current working directory to the directory where the script is located, and then looks for `filesystem.tar`.

The program checks for:

```bash
./filesystem.tar
```

If the archive is found, the program deletes the old `filesystem` directory and extracts a clean file system again. This resets item locations, permissions, and room states every time the game starts.

If `filesystem.tar` is not found but a `filesystem` directory already exists, the program can still continue running. However, this may prevent the game from functioning correctly. For example, the key may be placed inside the house, making it impossible for the player to enter the house.

---

## 3. Overall Implementation

### 1. Using Directories to Represent Rooms

Each room is a directory under `filesystem/pokey/rooms/`. The rooms are listed below:

```plain
bear-hangout
building-front
computer-room
dead-end
e-w-dirt-road
fork
hidden-area
mailroom
ne-sw-road
old-building-hallway
se-nw-road
```

Each room directory contains a `description` file, which stores the text description of that room.

One thing to note is that, in the original Dunnet, **the `mailroom` directory cannot be found when using pokey**. However, in this custom Dunnet implementation, it can be found.

```notion

                                         None
                                           A
                                           |n         
                                 w         |            e
                  computer-room <- old-building-hallway -> mailroom
                                          /
                                         /ne(s in reverse)
                                        /
                                 building-front
                                      /
                                     /ne
                                    /
                               ne-sw-road
                                  /
                                 /ne
         e                e     /
dead-end -> e-w-dirt-road -> fork
                                \
                                 \se
                                  \
                               se-nw-road
                                   \
                                    \se
                                     \
                                 bear-hangout
                                     /
                                    /sw
                                   /
                              hidden-area
```

### 2. Using Symbolic Links to Represent Movement Directions

If a room can be reached by moving east (`e`), then a corresponding symbolic link named `e` exists in the directory. This link points to the directory of the next room.

When the player enters a direction, the program first checks whether the corresponding symbolic link exists. If it exists, the program uses the `cd` command to enter the directory pointed to by the link.

Then, the program uses `pwd -P` to obtain the physical path after resolving the symbolic link. This is used to determine the player's current room.

All movement commands are listed below:

```bash
u
up

d
down

n
north

s
south

e
east

w
west

ne
northeast

nw
northwest

se
southeast

sw
southwest
```

### 3. Using `.o` Files to Represent Items

Interactive items are mainly represented by `.o` files, for example:

```notion
shovel.o
lamp.o
food.o
key.o
board.o
bracelet.o
paper.o
```

However, some `.o` files cannot be picked up, such as:

```notion
bear.o
boulder.o
```

When the player picks up an item, the program moves the corresponding file to:

```bash
filesystem/pokey/usr/toukmond
```

This directory represents both the player's inventory and the home directory after logging in to pokey.

### 4. Using Symbolic Links to Handle Item Aliases

Some items have multiple names. For example:

```plain
bracelet
emerald
```

Both refer to:

```plain
bracelet.o
```

```plain
board
card
chip
cpu
```

All refer to:

```plain
board.o
```

The program maps these names to the corresponding `.o` file name to prevent the same item from being treated as multiple independent items.

### 5. Using Timestamps to Control Output Order

When the player picks up or drops an item, the program executes:

```bash
touch "$object"
```

This updates the timestamp of the item.

Later, the program uses:

```bash
ls -tr
```

to select files according to their modification times, so that items can be displayed in interaction order when the player enters commands such as `get all`, `i`, and `l`.

---

## 4. Command Input

After the player enters a command, the program first reads the complete input:

```bash
set input = $<:q
```

Then it temporarily disables filename substitution:

```bash
set noglob
```

This prevents input such as:

```bash
x *
```

from being expanded by the outer shell into all files in the current directory.

### Case Handling

Input in normal game mode is case-insensitive:

```bash
set input = `echo "$input" | tr 'A-Z' 'a-z'`
```

For example:

```bash
GET SHOVEL
Get Shovel
get shovel
```

are all treated as:

```bash
get shovel
```

### Delimiter Handling

In Dunnet's normal adventure mode, the following characters are all treated as delimiters:

```bash
space
:
;
,
```

The program converts consecutive delimiters into a single space:

```bash
set input = `echo "$input" | sed 's/[ :;,][ :;,]*/ /g'`
```

Therefore, the following inputs have the same effect:

```bash
x shovel
x:shovel
x;shovel
x,shovel
x,:;;shovel
```

### Argument Array and Padding

After normalization, the program splits the input into an argument array:

```bash
set args = ( $input:x )
```

To avoid the `error: Subscript out of range` error when accessing nonexistent elements, the program appends several empty strings:

```bash
set args_padded = ( $args:q "" "" "" "" )
```

For example, if the player only enters:

```bash
get
```

The program can still safely access:

```bash
$args_padded[2]
$args_padded[3]
$args_padded[4]
```

---

## 5. Normal Game Mode Commands

| Command | Function | Implementation |
| --- | --- | --- |
| `n`, `s`, `e`, `w`, </br> `ne`, `nw`, `se`, `sw`, </br> `u`, `d` | Move to another room | Check whether a symbolic link with the same name exists in the current directory. If it exists, use `cd` to move, and use `pwd -P` to obtain the absolute path of the room. |
| `l`, </br> `look`, </br> `x`, </br> `examine`, </br> `read` | View detailed information about the current room | Read the `description` file in the current room, then read all `.o` files in the room, convert the file names into corresponding item descriptions, and output them. |
| `l <item>`, </br> `look`&nbsp;`<item>`, </br> `x`&nbsp;`item`, </br> `examine`&nbsp;`<item>`, </br> `read`&nbsp;`<item>` | Inspect a specified item. If no item name is entered, it outputs the current room information, equivalent to `l`. | According to the name entered by the player, use `cat` to output the text content of the corresponding `.o` file *(or ordinary file)*. |
| `i`, </br> `inventory` | View the items held by the player | List all `.o` files in `../../usr/toukmond/`, ignoring symbolic links. |
| `get <item>`, </br> `take <item>` | Pick up a specified item | Use `mv` to move the specified pickable item in the current room to `../../usr/toukmond/`, then use `touch` to update its timestamp. |
| `get all`, </br> `take all` | Pick up all pickable items in the room | Use `ls -tr *.o` to find items. Skip hidden items *(file names beginning with `.`)* and unpickable items, then move them to the inventory, `../../usr/toukmond/`, in order. If an item with symbolic links is picked up, such as `cpu` or `bracelet`, the corresponding symbolic links also need to be moved to the inventory. |
| `drop <item>`, </br> `throw <item>` | Drop an item | Use `mv` to move the item from the inventory to the current room, then use `touch` to update its timestamp. |
| `dig` | Dig the ground | First check whether the inventory contains `shovel.o`. If the player is in the specified room, `fork`, the hidden CPU board is revealed. |
| `put`&nbsp;`<item1>`&nbsp;`in`&nbsp;`<item2>`, </br> `insert`&nbsp;`<item1>`&nbsp;`in`&nbsp;`<item2>` | Put one item into another item | Currently mainly used to put the CPU board into the computer, which turns the computer on. The preposition is not strictly required to be `in`. |
| `type` | Operate the computer | Only valid in the `computer-room`. If the computer has been turned on, the player can enter the `pokey` login flow. |
| `exit`, `quit` | End the game | Terminate the program. |

---

## 6. Pokey — A Computer Simulating a UNIX Environment

### Login Method

After turning on the computer, enter:

```bash
type
```

The login screen will then appear.

The username and password can be obtained by checking the bin in the mailroom. They are:

```plain
login: toukmond
password: robert
```

After logging in successfully, the player enters the simulated UNIX environment:

```plain
UNIX System V, Release 2.2 (pokey)
```

Pokey's initial working directory is:

```bash
/usr/toukmond
```

### Pokey Path Handling

Pokey does not directly change the real shell's working directory to the virtual path. Instead, it uses:

```bash
# current directory in pokey, 
# initialized to "/usr/toukmond" since it's the home directory of toukmond
set pokey_pwd = "/usr/toukmond"
```

to record the player's location in Pokey.

The program then converts the Pokey path to a location in the real file system:

```bash
$game_root/filesystem/pokey
```

When handling `cd` and `ls`, the program temporarily uses the real `cd` and `pwd -P` to normalize paths, including:

```bash
.
..
repeated /
```

It also prevents the player from leaving the Pokey root.

```bash
# pokey root directory in real filesystem, 
# used for path normalization in cd and ls command in pokey
set pokey_root = "$game_root/filesystem/pokey" 
```

---

## 7. Pokey Commands

| Command | Function | Implementation |
| --- | --- | --- |
| `exit` | Leave Pokey and return to normal adventure mode | Display the console exit message and jump back to `game_loop`. |
| `echo <text>` | Display the entered text | Output everything after the second argument. |
| `pwd` | Display Pokey's current path | Display `pokey_pwd`. |
| `ls` | Display the contents of the current directory | Display predefined UNIX-like output according to `pokey_pwd` *(hard-coded)*, and dynamically list `.o` files in the inventory or room. |
| `ls <path>` | Display the contents of a specified directory | First normalize the relative or absolute path, then display the corresponding directory. |
| `cd <path>` | Change Pokey's working directory | Normalize the target path and update `pokey_pwd`. This does not modify the real working path of the game itself. |
| `cat <file>` | Display the contents of an ASCII file | Only allows reading certain text files in the current directory, such as `description`. |
| `uncompress` | Decompress a file | Use `gunzip` to decompress the specified file. The currently decompressible file in the game is `paper.o.gz` *(in the original Dunnet, it is `paper.o.Z`)*. |
| `ftp` | *Not completed* | *Not required by the assignment; this function has not been implemented.* |
| `rlogin` | *Not completed* | *Not required by the assignment; this function has not been implemented.* |
| `ssh` | *Not completed* | *Not required by the assignment; this function has not been implemented.* |

### Input Differences Between Pokey and Normal Game Mode

Normal adventure mode treats:

```bash
:
;
,
```

as delimiters.

Pokey simulates a UNIX shell, so it currently splits arguments only by spaces. The delimiter rules used in normal game mode should not be directly applied to Pokey.

---

## 8. Problems Encountered During Development and Solutions

### 1. `*` Being Expanded as a Wildcard

### Problem

When the player enters:

```bash
x *
```

`*` may be treated by the outer shell as a filename glob pattern and expanded into the file names in the current room.

For example, if the room contains:

```plain
lamp.o
shovel.o
```

Entering:

```bash
x *
```

may be interpreted as:

```plain
x lamp.o shovel.o
```

This may cause the program to incorrectly display the information of an item.

### Solution

Before parsing input in normal game mode, add:

```bash
set noglob
```

Then restore the original state after creating `args_padded`:

```bash
unset noglob
```

The `noglob` variable in `tcsh` disables filename substitution, so:

```bash
*
?
[abc]
```

will not be expanded into file names by the shell.

---

### 2. Dunnet Uses Multiple Delimiters

### Problem

In the original Dunnet, spaces, colons, semicolons, and commas are all treated as argument delimiters.

Therefore:

```bash
x shovel
x:shovel
x;shovel
x,shovel
```

should have the same effect.

### Solution

Use `sed` to convert consecutive delimiters into a single space:

```bash
set input = `echo "$input" | sed 's/[ :;,][ :;,]*/ /g'`
```

---

### 3. Empty Commands Causing Errors

### Problem

In normal game mode or Pokey, if the player does not enter any content and only presses Enter, the program may produce errors while parsing arguments or accessing array elements.

### Solution

Use the following when reading input:

```bash
# normal input
set input = $<:q

# pokey input
set pokey_cmd = $<:q
```

Then, before accessing arguments, check the array length:

```bash
if ( "$#args" == 0 ) goto game_loop

if ( "$#pokey_args" == 0 ) goto pokey_loop
```

`:q` reduces the chance that input will be reinterpreted by the shell. The array length checks skip truly empty commands.

These two mechanisms serve different purposes, and both are necessary.

---

### 4. Accessing Nonexistent Array Elements

### Problem

In `csh`, directly accessing a nonexistent array element, such as:

```bash
$args[3]
```

may produce: `error: Subscript out of range`

### Solution

Create a padded array:

```bash
set args_padded = ( $args:q "" "" "" "" )
```

Then always access:

```bash
$args_padded[1]
$args_padded[2]
$args_padded[3]
$args_padded[4]
```

Pokey uses the same method:

```bash
set pokey_args_padded = ( $pokey_args:q "" "" "" "" )
```

---

### 5. Inconsistent Room Paths Caused by Symbolic Links

### Problem

After the player moves through a directional link, the logical path and physical path may differ.

For example:

```bash
e -> ../e-w-dirt-road
```

If the currently displayed path is used directly to determine the room, the visited-room detection may become incorrect.

### Solution

After movement, use:

```bash
pwd -P
```

to obtain the physical path after resolving symbolic links.

Pokey's `cd` and `ls` also normalize paths by temporarily switching to the real directory and then using:

```bash
pwd -P
```

---

### 6. One Item Having Multiple Names

### Problem

The CPU board may be referred to as:

```plain
board
card
chip
cpu
```

The bracelet may also be referred to as:

```plain
bracelet
emerald
```

If each name is treated as an independent item, the player may be able to pick up the same item multiple times.

### Solution

Use one main `.o` file to represent the actual item:

```plain
board.o
bracelet.o
```

---

## 9. Unhandled or Unconfirmed Issues

### 1. `ls` in Pokey Displays `boulder.o`

After logging in to Pokey, switch to:

```bash
/rooms/e-w-dirt-road
```

Then enter:

```bash
ls
```

The current version displays:

```plain
boulder.o
```

However, the original Dunnet does not display this file.

### Cause

The current Pokey `ls` logic lists all ordinary `.o` files in a room directory:

```bash
foreach file ( `sh -c "ls -tr $game_root/filesystem/pokey/${target_path}/*.o 2>/dev/null"` )
```

Since `boulder.o` meets this condition, it is output.

### Items to Be Confirmed

The handling method used by the original Dunnet has not yet been confirmed. Possible reasons include:

1. The original version additionally filters out unpickable items.
2. `boulder` is not an ordinary `.o` file in the original version.
3. The original version's room `ls` uses another whitelist-based logic.
