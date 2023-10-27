# `Crackme3` Project — ALX (Holberton School)
I've decided to write this guide because I've noticed that numerous ALX Software Engineering (SE) students often seek advice on how to tackle the crackme3 project. ALX often assigns advanced tasks at the end of each curriculum or project, and crackme3 is part of the 'Bit Manipulation' class, which can be quite challenging. In this guide, I'll break down the project step by step to make it easier to understand.

## What we know:
Before we dig into the process of decoding our file. Let's see what information we have:\
***Crackme3*** file is our target file. the password is expected to be provided as an argument on the command line.
```properties
$ ./crackme3 `cat 101-password`
ko 😥
```
if we run the "file" command on our crackme this the result:
```properties
$ file crackme3
crackme3: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=b563744686f66961ad9d58dccf332bc11c01cf57, not stripped
```
The **file** command output reveals that 'crackme3' is a 64-bit ELF (Executable and Linkable Format) binary executable. What's interesting is that it's labeled as '**not stripped**' signifying the presence of debugging symbols within the binary. This can be advantageous for debugging and analysis purposes, as it provides valuable insights into the program's internal workings.

I was wrong! At first, I tried opening the 'Crackme3' file in **Vim** to examine its source code, hoping to understand how the password is generated and verified. But it was not the case, Crackme3 is an executable file !! so we can't see it's source code in the way we used to open ***.c*** or ***.sh*** files.

I did some researches here and there and I discover that I should use some reverse engineering techniques!

## Reverse Engineering:
Reverse engineering is the art of taking things apart to understand how they work, often used in software, hardware, and technology to learn, modify, or enhance existing systems.
## How it is done?
1. Starts by observing and studying the system or software we want to reverse engineer.
2. Take detailed notes and document your observations and findings.
3. Use tools like a disassembler or debugger to examine the program's assembly code.
4. Analyze the program's logic and identify the part we are interested in, then try to reverse engineer the logic to reach our goal.

## In Action:
First I want you to take a look at these four common Linux tools that widly used in reverse engineering:

```Strings```: The "strings" command in Linux can reveal plaintext (passwords) hidden in binary files. It searches for sequences of characters that resemble text. We may use this tool to scan executables for any password-related strings left in plain sight.

```Objdump```: "Objdump" is a powerful debugging and disassembling tool. It can be used to analyze the assembly code of a program, which might reveal how passwords are processed and stored in memory.

```Ltrace```: Ltrace is a dynamic library call tracer. It tracks library calls made by a program, which can include password-related functions. We may use Ltrace to monitor a program's behavior, such as the functions called during password validation, potentially revealing clues about the password-handling process.

```GDB (GNU Debugger)```: It is the most powerful tool in this list, GDB is a debugger that allows you to step through and analyze a program's execution. It can be used to inspect the program's memory, which might contain valuable data.

<hr>

Now we have the tools needed to decode our Crackme file, let's try them!
```properties
$ strings ./crackme3 password
/lib64/ld-linux-x86-64.so.2
libc.so.6
puts
strlen
stderr
fprintf
__libc_start_main
__gmon_start__
GLIBC_2.2.5
UH-H
AWAVA
AUATL
[]A\A]A^A_
Usage: %s password
Congratulations!
;*3$"
# more ...
```
As you can see ```strings``` command writes every string of printable text from within a binary file to the standard output.\
If the crackme3's developer embeds a plaintext password within the executable file, "strings" command could easily extract the password.

Let's try ```ltrace``` command and see what information could we get
```properties
$ ltrace ./crackme3 password
__libc_start_main(0x400675, 2, 0x7ffccef78948, 0x4006f0 <unfinished ...>
strlen("password")                                                                              = 8
puts("ko"ko
)                                                                                      = 3
+++ exited (status 1) +++
```
As you can see we got a useful information, running the "Ltrace" command shows that the program is checking for the **length** of the password we have passed. but there is no clear indication how the program works.

From the previous attempts we can conclude that knowing how the Crackme3 works wasn't that easy 😥
So, we must procced to a more advanced technique!

### Using the GNU debugger:
GDB is a tool developed for Linux systems with the goal of helping developers identify sources of bugs in their programs.

```console
GDB, the GNU Project debugger, allows you to see what is going on `inside’ another program while it executes — or what another program was doing at the moment it crashed.
```

When engaging in reverse engineering, GDB is employed to examine the compiled Assembly code using either AT&T or Intel flavor (syntax). This allows for a detailed, step-by-step inspection of the program's inner workings. Breakpoints are strategically inserted to pause the program's execution, enabling an in-depth review of data in memory registers and facilitating the identification of data manipulation processes.

To get started, I launch the crackme3 file with GDB followed by the following commands, This will allow us to view the assembly code of the "main" function in the "crackme3" executable using the AT&T syntax which making it human-readable for analysis.

```console
gdb ./crackme3
set disassembly-flavor att
disassemble main
```
Result:
![disassemble main - AT&T](imgs/disessemble%20main%20-%20AT&T.png)

The first column provides the address of the command. The next column is the command itself followed by the data source and the destination.

While it may seem daunting, understanding the inner workings of the program step by step is achievable, even if you're not well-versed in assembly language. A practical approach involves focusing on commands that may leak data, like those related to printing or comparing data. For instance, commands like '**fprintf@plt**' and '**puts@plt**' are familiar C functions that likely output something to the standard output. Ultimately, our primary objective is to uncover the password. The crucial piece of the puzzle is the '**check_password**' function, as it plays a pivotal role in determining the program's actions. To decipher what happens inside this function, we executed the following command.
```sh
disassemble check_password
```

Result:
![disassemble check_password - AT&T](imgs/disassemble%20check_password.png)

At command address 0x400609, the program calls the **strlen** function, which calculates the length of a string and stores the result in the **rax** register. Subsequently, it compares the value in the **rax** register to the hexadecimal value 0x4 (equivalent to 4 in decimal). This comparison aims to determine whether the result from the **strlen** function is equal to 4.\
Now that we know the password's length, the program proceeds to perform a comparison at command address 0x400653. This comparison involves two registers, namely, **rax** and **rdx**

> **rax** and **rdx**: general-purpose registers in Assembly language for storing and manipulating data during program execution.

What we can do to investigate that comparison, is to set a **breakpoint** at that command address and run our program
> A breakpoint is point where the program execution process is interrupted. so we can see realtime execution data.

```console
(gdb) break *0x400653
```

By setting a breakpoint at '0x400653,' we ran the program while providing a four-character password (in this case, '**test**') to bypass the length condition. This allowed us to reach our breakpoint, where the program compares data. At this critical juncture, we use the '**info registers**' command to examine the program's registers.

```console
(gdb) run test
(gdb) info registers
```

Result:
![Alt text](<imgs/info registers.png>)

In the registers data snapshot, we observed the values stored in both **rax** and **rdx**, which are being compared within the **check_password** function. The first value, 0x74, corresponds to the character **‘t’**, while the second value, 0x48, represents the character **‘H**’.

The program iterates through the password, comparing each character. Notably, **rax** contains characters from our input password, while **rdx** holds the actual correct password. 

Attempting to proceed to the next loop using the **continue** GDB command led to the program's exit due to the **check_password** function terminating when it encounters a character mismatch.

#### So, How can we crack other password characters?

We already have the first password character, Right? So, what we can do is simply replace it in the password that we pass to the program. exemple:

```console
(gdb) run Hest
```
We used to provide "test" as a password, and since we know the first character **H** we just replaced it in the input password. This enabled us to use ```continue``` to procced to the next loop where the program compares the next character of the password. \
So, we will keep reveiling password characters one by one. everytime we reaveil a character we feed it to the program in a new run.\
Keep in mind that our breakpoint that we set is located inside a loop, so the program will stops on each loop to compare next characters. and ofcourse this is our chance to examine the registers with "**info integers**" command.\
At the end, this will be the final result:
```js
// the value of rdx register on each loop
LOOP_1: rdx - 0x48 => H
LOOP_2: rdx - 0x6f => o
LOOP_3: rdx - 0X6C => l
LOOP_4: rdx - 0x4  => ^D // (End of file)
```

### We are successfully 🎉 cracked the password = ```"Hol^D"```

But, we are not done yet? We are required to save the password in a ```101-password``` and feed it to the **crackme3**\
we can't just use the following command to write the password in the text file.

```sh
echo "Hol^D" > 101-password ❌
```

**Why?** Bash interprets the EOF file command so it isn’t passed to the executable. In fact, it is used to exit executables. and if we tried to store it in a file, ***Vim*** / ***Emacs*** could reads ^D as an end of buffer command.

Inputting the password poses a bit of a challenge! So, our workaround would be .. 

```sh
echo -n -e "Hol\x04" > 101-password
```

"**-n**": This option tells echo not to print a trailing newline character, so the output doesn't end with a newline.

"**-e**": This option enables the interpretation of backslash escapes, such as \x04, which is used to represent a control character.

"**Hol\x04**": This is the text that will be echoed. It consists of the string "Hol" followed by the control character ^D (EOF, represented as \x04 in hexadecimal).

```properties
$ ./crackme3 `cat 101-password`
Congratulations! 🥳
```

### Note:
As previously noted, during each loop, **rax** contains a character from our input password, and **rdx** holds a character from the true password. This raises the question: is the entire password stored somewhere in the program?

Upon examining the '**info registers**' result more closely, we notice another register named **rcx** which holds the value **0x46c6f48**. Comparing this value with the characters we've extracted in each loop, we can infer that it contains the entire password in reverse order. Reading it in reverse yields **48**, **6f**, **6C**, and **4**, which, when converted to character values, correspond to **H**, **o**, **l** and **^D** or EOF

![rcx register value](<imgs/rcx register value.png>)

## Conclusion:
The primary aim of this reverse engineering project is to help students grasp how software works internally and the basics of security. By examining and breaking down code, students gain insights into program structures, vulnerabilities, and the importance of strong security practices. These projects promote a well-rounded understanding of software security, emphasizing ethical hacking and effective defense strategies, preparing students for future roles in tech with a security focus.

## References:
[**Reverse-engineering: Using Linux GDB**](https://medium.com/@rickharris_dev/reverse-engineering-using-linux-gdb-a99611ab2d32) — *Rick Harris*

[**4 Ways a Password Could be Hacked Using Common Linux Tools**](https://www.linux.com/training-tutorials/4-ways-password-could-be-hacked-using-common-linux-tools/) — *Rick Harris*


## Author:
**Ouadia EL-Ouardy** — [*https://github.com/EL-OUARDY*](https://github.com/EL-OUARDY)


