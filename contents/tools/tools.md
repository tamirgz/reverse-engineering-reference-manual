# .tools

## *<p align='center'> IDA Tips </p>*
* __Addresses Shown In IDA__: when IDA loads a binary, it simulates a mapping of the file in memory. The addresses shown in IDA are the virtual memory addresses and not offsets of the binary file on disk

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/tools/IDA_Tips/ida_va_instr.PNG"> 
<p align='center'><sub><strong>IDA displaying 4 instructions along with their respective virtual addresses</strong></sub></p>
</div>

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/tools/IDA_Tips/ida_va_hex.PNG"> 
<p align='center'><sub><strong>IDA displaying those 4 instructions in hex. Note that the virtual addresses are the same</strong></sub></p>
</div>

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/tools/IDA_Tips/hex_on_disk.PNG"> 
<p align='center'><sub><strong>Actual locations of those 4 instructions on disk</strong></sub></p>
</div>

* __Import Address Table (IAT)__: shows you all the dynamically linked libraries' functions that the binary uses. IAT is important for a reverser to understand how the binary will be interacting with the OS. To hide API calls from displaying in the IAT, a programmer can dynamically resolve them
  + __How To Find Dynamically Resolved APIs__: get the binary's function trace (e.g. hybrid-analysis, ltrace). If any of the APIs in the function trace is not in the IAT, then that API is dynamically resolved
  * __How To Find Where A Dynamically Resolved API Is Called__: in IDA's debugger view, the Module Windows allows you to place a breakpoint on any function in a loaded dynamically linked library. Use it to place a breakpoint on a dynamically resolved API and once execution breaks there, step back through the call stack to find where it's called from in user code

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/tools/IDA_Tips/source.png" width="500" height="430">
<p align='center'><sub><strong><a href="https://gist.github.com/yellowbyte/ec470d75ba7c14ebefed271c6fe58e9e">source code</a> showing how `puts` is dynamically resolved. String reference to `puts` is also encoded</strong></sub></p>
</div>

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/tools/IDA_Tips/iat.png" width="470" height="370">
<p align='center'><sub><strong>even though `puts` is a function from a dynamically linked library it does not show up in IDA's IAT</strong></sub></p>
</div>

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/tools/IDA_Tips/strings.png" width="500">
<p align='center'><sub><strong>GNU strings can't identify string reference to `puts` either</strong></sub></p>
</div>

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/tools/IDA_Tips/ltrace.png" width="500">
<p align='center'><sub><strong>function tracer like ltrace is able to detect reference to `puts`</strong></sub></p>
</div>

* __To Save Memory Snapshot From Your Debugger Session__: Debugger -> Take Memory Snapshot -> All Segments
* __Useful Shortcuts__: 
  + __u__ to undefine 
  + __d__ to turn it to data 
  + __c__ to turn it to code 
  + __g__ to bring up the jump to address menu
  + __n__ to rename
  + __x__ to show cross-references
#
## *<p align='center'> GDB Tips </p>*
* __Changing Default Settings__: to make changes permanent, write it in the .gdbinit file
  * ASLR is turned off by default in GDB. To turn it on: __set disable-randomization off__
  * Default displays assembly in AT&T notation. To display assembly in Intel notation: __set disassembly-flavor intel__ 
* __User Inputs__: how to pass user inputs to debugged program as arguments or/and as stdin
  * After starting GDB...
      ```gdb
      (gdb) run argument1 argument2 < file
      ```
    * content of file will be passed to debugged program's stdin
* __Automation__: ways to automate tasks in GDB
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
* __Ways To Pause Debuggee__: software breakpoint, hardware breakpoint, watchpoint, and catchpoint
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
* __Useful Commands__: apropos, i, x, and set
  * apropos &lt;arg&gt; command searches through all gdb commands/documentations for &lt;arg&gt; and displays matched command/documentation pairs  

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/tools/GDB_Tips/apropos_ex.png" width="600">
<p align='center'><sub><strong>gdb output from 'apropos mapping'</strong></sub></p>
</div>

  * i command displays information on the item specified to the right of it
    + __i proc mappings__: shows mapped address spaces 
    + __i b__: shows all breakpoints 
    + __i r__: shows the values in general purpose, flag, and segment registers at that point of execution
    + __i all r__: shows the values in all registers at that point of execution, such as FPU and XMM registers  
  * x command displays memory contents at a given address in the specified format
    + Since disas command won't work on stripped binary, x command can come in handy to display instructions from current program counter: __x/14i $pc__
  * set command sets temporary variable: __set $&lt;variable name&gt; = &lt;value&gt;__
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
