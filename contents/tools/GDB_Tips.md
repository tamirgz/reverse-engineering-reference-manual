### [.tools](tools.md)[__GDB Tips__]

---
#### *<p align='center'> Changing Default Settings </p>*
---
* To make changes permanent, write it in the .gdbinit file
* ASLR is turned off by default in GDB. To turn it on: __set disable-randomization off__
* Default displays assembly in AT&T notation. To display assembly in Intel notation: __set disassembly-flavor intel__ 

---
#### *<p align='center'> User Inputs </p>*
---
* How to pass user inputs to debugged program as arguments or/and as stdin
  * After starting GDB...
      ```gdb
      (gdb) run argument1 argument2 < file
      ```
    * content of file will be passed to debugged program's stdin

---
#### *<p align='center'> Automation </p>*
---
* __-x Option__: puts the list of commands you want GDB to run when gdb starts in a file. Run GDB with the -x option like this:
    ```bash
    gdb -x command_file program_to_debug
    ```
* __Hooks__: user-defined command. When command ? is ran, user-defined command 'hook-?' will be executed (if it exists)
  + When reversing, it could be useful to hook on breakpoints by using hook-stop 
  + How to define a hook: 
     ```gdb
     (gdb) define hook-?
     > ...commands...
     > end
     (gdb)
     ```

---
#### *<p align='center'> Ways To Pause Debuggee </p>*
---
* __Software Breakpoint__:
  ```bash
  (gdb) break *0x8048479
  ```
  * __shortcut__: if the instruction pointer is at the address that you wanted to break at, simply type b or break and a breakpoint will be set there
* __Hardware Breakpoint__:
  ```bash
  (gdb) hbreak *0x8048479 
  ```
* __Watchpoint__:
  ```bash
  (gdb) watch *0x8048560  #break on write
  (gdb) rwatch *0x8048560 #break on read
  (gdb) awatch *0x8048560 #break on read/write
  ```
* __Catchpoint__:
  ```bash
  (gdb) catch syscall #break at every call/return from a system call
  ```

---
#### *<p align='center'> Useful Commands </p>*
---
* __apropos &lt;arg&gt;__ command searches through all gdb commands/documentations for &lt;arg&gt; and displays matched command/documentation pairs  
<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/tools/GDB_Tips/apropos_ex.png" width="600">
<p align='center'><sub><strong>gdb output from 'apropos mapping'</strong></sub></p>
</div>

* __i (info)__ command displays information on the item specified to the right of it
  + __i proc mappings__: shows mapped address spaces 
  + __i b__: shows all breakpoints 
  + __i r__: shows the values in general purpose, flag, and segment registers at that point of execution
  + __i all r__: shows the values in all registers at that point of execution, such as FPU and XMM registers  
* __x (examine)__ command displays memory contents at a given address in the specified format
  + Since disas command won't work on stripped binary, x command can come in handy to display instructions from current program counter: __x/14i $pc__
* __set__ command sets temporary variable: __set $&lt;variable name&gt; = &lt;value&gt;__
  * set command can be used to change the flags in the EFLAGS register. You just need to know the bit position of the flag you wanted to change 
  + To set the zero flag:
    ```bash
    (gdb) set $ZF = 6                #bit position 6 in EFLAGS is zero flag
    (gdb) set $eflags |= (1 << $ZF)  #use that variable to set the zero flag bit
    ```
<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/tools/GDB_Tips/eflags.png" width="600" height="120">
<p align='center'><sub><strong>each available flag and its corresponding bit position in the EFLAGS register</strong></sub></p>
</div>

#
<p align='center'><a href="IDA_Tips.md">IDA_Tips</a> <~ <a href="tools.md">.tools</a> ~> <a href="/contents/instruction-sets/instruction-sets.md">.instruction-sets</a></p>
