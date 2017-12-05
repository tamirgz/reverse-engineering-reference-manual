# <p align='center'> Reverse Engineering Reference Manual </p>

<p align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/heading/Introduction.PNG"> 
</p>

# .table-of-contents

* [.general-knowledge](#general-knowledge)
  + [int 0x7374617274](#-int-0x7374617274-)
* [.tools](#tools)
  + [IDA Tips](#-ida-tips-)
  + [GDB Tips](#-gdb-tips-)
  + [WinDBG Tips](#-windbg-tips-)
* [.instruction-sets](#instruction-sets)
  + [x86](#-x86-)
  + [x86-64](#-x86-64-)
  + [ARM](#-arm-)
* [.languages](#languages)
  + [C++ Reversing](#-c-reversing-)
  + [Python Reversing](#-python-reversing-)
* [.file-formats](#file-formats)
  + [ELF Files](#-elf-files-)
  * [PE Files](#-pe-files-)
* [.anti-analysis](#anti-analysis)
  + [Obfuscation](#-obfuscation-)
  + [Anti-Disassembly](#-anti-disassembly-)
  + [Anti-Debugging](#-anti-debugging-)
  + [Anti-Emulation](#-anti-emulation-)
  + [Bonus](#-bonus-)
* [.encodings](#encodings)
  + [String Encoding](#-string-encoding-)
  + [Data Encoding](#-data-encoding-)
---

__NOTE__: from now until the end Jan 2018, I am planning on adding more pics/diagrams to go along with the notes to make them more understandable and easier to read. Stay tuned!

# .general-knowledge

## *<p align='center'> int 0x7374617274 </p>*
* __Threads__: a process is a container for execution. A thread is what the OS executes
  * A process that doesn't utilizes multi-threading still contains a single thread

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/general-knowledge/int_0x7374617274/4_01_ThreadDiagram.png" width="469" height="270">
<p align='center'><sub><strong>single threaded vs multi-threaded process</strong></sub></p>
</div>

* __start Is Not main__: entry point of a binary (start function) is not main. A program's startup code (how main is set up and called) depends on the compiler and the platform that the binary is compiled for
  * Even if no library is statically compiled into the binary, part of the .text section will contain code that is irrelevant to the source code

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/general-knowledge/int_0x7374617274/start_v_main.PNG"> 
<p align='center'><sub><strong>32-bit ELF binary compiled by gcc</strong></sub></p>
</div>

* __Random Number Generator__: Randomness requires a source of entropy (seed), which is an unpredictable sequence of bits that can come from the OS observing its internal operations or ambient factors. Algorithms using a seemingly unpredictable sequence of bits as seed are known as pseudorandom generators, because while their output isn't random, it still passes statistical tests of randomness
* __Software Breakpoint__: debugger reads and stores the first byte of instruction and then overwrites that first byte with 0xCC (INT3). When CPU hits the breakpoint (0xCC), OS kernel sends SIGTRAP signal to process, process execution is paused by the debugger, and debugger's internal lookup occurs to flip the original byte back

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/general-knowledge/int_0x7374617274/soft_bp.png"> 
<p align='center'><sub><strong>software breakpoint</strong></sub></p>
</div>

* __Hardware Breakpoint__: set at CPU level in special registers called debug registers
  * Debug registers (DR0 through DR7)
    * DR0-DR3: stores addresses of hardware breakpoints
    * DR4-DR5: reserved
    * DR6: status register that contains information on which debugging event has occurred
    * DR7: stores breakpoint conditions and the lengths of breakpoints for DR0-DR3
  + Before CPU attempts to execute an instruction, it first checks whether the address is currently enabled for a hardware breakpoint. If the address is stored in debug registers DR0–DR3 and the read, write, or execute conditions are met, an INT1 is fired and the process halts
  + Can check if someone sets a hardware breakpoint on Windows by using GetThreadContext() and checks if DR0-DR3 is set

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/general-knowledge/int_0x7374617274/hardware_bp.png" width="469" height="270"> 
<p align='center'><sub><strong>hardware breakpoint</strong></sub></p>
</div>

* __Memory Breakpoint__: changes the permissions on a region, or page, of memory
  + Guard page: Any access to a guard page results in a one-time exception, and then the page returns to its original status. Memory breakpoint changes permission of the page to guard

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/general-knowledge/int_0x7374617274/memory_bp.png"> 
<p align='center'><sub><strong>memory breakpoint</strong></sub></p>
</div>

* __Endianness__: Intel x86 and x86-64 use little-endian format. It is a format where multi-bytes datatype such as integer has its least significant byte stored in the lower address of main memory. Due to Intel's endianness, here are points to keep in mind when reversing: 
  * Characters or user input, whether constructed on stack or already initialized and placed in data section, will be placed in memory in the order that it comes in since each character is a byte long, so endianness doesn't matter. But if you read out multi-characters at a time, such as 4 characters using DWORD[addr], CPU will think that the 4 bytes at addr are in little-endian and will then retrieve those bytes in reverse order 
  * Multi-bytes datatype, such as integer, will be stored in little-endian in memory. But when accessed from memory, it will be in its original form since CPU assumes little-endian
  * Value stored in RAM is in little-endian but when moved to a register it is in big-endian
---

# .tools

## *<p align='center'> IDA Tips </p>*
* __Addresses Shown In IDA__: When IDA loads a binary, it simulates a mapping of the file in memory. The addresses shown in IDA are the virtual memory addresses and not the offsets of binary file on disk

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

* __Import Address Table (IAT)__: shows you all the dynamically linked libraries' functions that the binary uses. IAT is important for a reverser to understand how the binary is interacting with the OS. To hide APIs call from displaying in the IAT, a programmer can dynamically resolve an API call
  + __How To Find Dynamically Resolved APIs__: get the binary's function trace (e.g. hybrid-analysis (Windows sandbox), ltrace). If any of the APIs in the function trace is not in the IAT, then that API is dynamically resolved. Once you find a dynamically resolved API, you can place a breakpoint on the API in IDA's debugger view (go to Module Windows, find the shared library the API is under, click on the library and another window will open showing all the available APIs, find the API that you are interested in, and place a breakpoint on it). Once execution breaks there, step back through the call stack to find where it's called in user code
  * It is considered normal for functions to appear in just the IAT and not in the function trace since function trace might not hit every single execution path. Through smart fuzzing, function trace coverage can be improved
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
  * Default displays assembly in AT&T notation. To change it to the more readable and superior Intel notation: __set disassembly-flavor intel__ 
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
* __Ways To Pause Debugger__: software breakpoint, hardware breakpoint, watchpoint, and catchpoint
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
* __Useful Commands__: i, x, p, and set
  * i command displays information on the item specified to the right of it
    + __i proc mappings__: show mapped address spaces 
    + __i b__: show all breakpoints 
    + __i r__: show the values in registers at that point of execution
  * x command displays memory contents at a given address in the specified format
    + Since disas command won't work on stripped binary, x command can come in handy to display instructions from current program counter: __x/14i $pc__
  * p command displays value stored in a named variable
  * set command sets temporary variable: __set &lt;variable name&gt; = &lt;value&gt;__
    * set command can be used to change the flags in the EFLAGS register. You just need to know the bit position of the flag you wanted to change 
    + To set the zero flag:
      ```bash
      (gdb) set $ZF = 6                #bit position 6 in EFLAGS is zero flag
      (gdb) set $eflags |= (1 << $ZF)  #use that variable to set the zero flag bit
      ```
    + To figure out the bit position of a flag that you are interested in:
    
<p align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/tools/GDB_Tips/eflags.png" width="600" height="120"> 
</p>

#
## *<p align='center'> WinDBG Tips </p>*
* __WinDBG Notations__: 
  * __?__: evaluates an expression 
  * __??__: evaluates a C++ expression
  * __!__: prefixed to tell debugger that the token is a symbol and not an expression
    * symbol can also be prefixed with module name (e.g. &lt;module&gt;!&lt;symbol&gt;) to save debugger time from searching through all modules for the matching symbol
  * __$__: prefixed to all pseudo-registers (e.g. $ip, $peb) 
  * __@__: prefixed to tell debugger that the token is a register or pseudo-register to save it time from doing symbol lookup
* __Ways To Find Entry Point__: WinDBG will break in the kernel, not at the entry point
  * Use __lm (loaded modules)__ to find the binary's image base. From image base, here are 2 ways to get entry point: 
    * __!dh (dump headers)__ to read the image's header information to get the entry point
    * __$iment__ to display just the entry point and not the rest of the header information 
  * __$exentry__: a pseudo-register that contains the entry point
* __Useful Commands__: 
  * __poi(address)__: displays data pointed to by address
  * __d[b/w/d/q/yb/a/u/f/D/p] address L&lt;num&gt;__: displays memory. The num right next to L is the range specifier that specifies the amount to display
    * dd deadbeef L4 will display 4 4-bytes values starting from address deadbeef 
  * __e[b|d|D|f|p|q|w] address [values]__: edits memory
    * ed deadbeef 0x10101010 0x20202020 will replace 2 4-bytes values starting from address deadbeef with 0x10101010 and then 0x20202020
  * __~__: lists all threads. ~Ns switches to the Nth thread
  * __|__: lists current process and all child processes. |Ns switches to the Nth process
  * __sx(e/d/r/i)__: controls how the debugger handle exceptions or events
    * __sxe__: breaks on an event 
    * __sxd__: disables break for an event 
    * __sxr__: shows output for an event 
    * __sxi__: ignores an event 
---

# .instruction-sets

## *<p align='center'> x86 </p>*
* __Registers__: temporary storage locations that is built into the CPU. Aside from the General Purpose Registers (GPRs), most other registers are dedicated to a specific purpose
  * The 6 32-bit selector registers for x86 architecture: CS, DS, ES, FS, GS, SS. A selector register contains address to a specific block of memory from which one can read or write. The real memory address is looked up in an internal CPU table 
    + Selector registers usually points to OS specific information. For example, FS segment register points to the beginning of current Thread Environment Block (TEB), also know as Thread Information Block (TIB), on Windows. Offset zero in TEB is the head of a linked list of pointers to exception handler functions on 32-bit system. Offset 30h is the Process Environment Block (PEB) structure. Offset 2 in the PEB is the BeingDebugged field. In x64, PEB is located at offset 60h of the gs segment register
  * Control register: EFLAGS. EFLAGS is a 32-bit register. It contains values of 32 boolean flags that indicate results from executing the previous instruction. EFLAGS is used by JCC instructions to decide whether to jump or not
* __How EIP Can Be Updated__: CALL, JMP, or RET
  * __Calling Conventions (x86)__: how function call is set up
    + __CDECL__: arguments pushed on stack from right to left. Caller cleaned up stack after
    + __STDCALL__: arguments pushed on stack from right to left. Callee cleaned up stack after
    + __FASTCALL__: first two arguments passed in ECX and EDX. If there are more, they are pushed onto the stack
    * The call instruction contains a 32-bit signed relative displacement that is added to the address immediately following the call instruction to calculate the call destination
  * __Jump Instruction__: 
    * __Short (Near) Jump__: like call instruction uses relative addressing, but with only an 8-bit signed relative displacement
    * __Long (Near) Jump__: uses larger offset value but also uses relative addressing from instruction pointer
    * __Far Jump__: uses absolute addresssing to jump to a location in a different segment. Needs to specify the segment to jump to and also the offsets from that segment 
    * x86 instruction set does not provide EIP-relative data access the way it does for control-flow instructions. Thus to do EIP-relative data access, a general-purpose register must first be loaded with EIP
* __Assembly to Machine Code Is Not One-To-One__: an opcode can have multiple mnemonics associated with it and a mnemonic can have multiple opcodes associated with it
  * __Example 1__: 0x75 is both the opcode for JNZ and JNE
  * __Example 2__: 0xb142 and 0xc6c142 both corresponds to the instruction MOV CL, 66
* __Lost Of Type Information__: there is no way to tell the datatype of something stored in memory by just looking at the location of where it is stored. The datatype is implied by the operations that are used on it. For example, if an instruction loads a value into EAX, comparison is taken place between EAX and 0x10, and JA is used to jump to another location if EAX is greater, then we know that the value is an unsigned int since JA is for unsigned numbers
* __Floating Point Arithmetic__: Floating point operations are performed using the FPU Register Stack, or the "x87 Stack." FPU is divided into 8 registers, st0 to st7. Typical FPU operations will pop item(s) off the stack, perform on it/them, and push the result back to the stack
  + FLD instruction is for loading values onto the FPU Register Stack
  + FST instruction is for storing values from ST0 into memory 
  + FPU Register Stack can be accessed only by FPU instructions
* __Variable-Length Instruction__: an instruction's size in x86 can range from 1 to 15 bytes

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/instruction-sets/x86/x86.png" height="600" width="400"> 
<p align='center'><sub><strong>one byte x86 instructions</strong></sub></p>
</div>

* __Commonly Used Hard To Remember x86 Instructions With Side Effects__:
  * __IMUL reg/mem__: register is multiplied with AL, AX, or EAX and the result is stored in AX, DX:AX, or EDX:EAX
  * __IDIV reg/mem__: takes one parameter (divisor). Depending on the divisor’s size, div will use either AX, DX:AX, or EDX:EAX as the dividend, and the resulting quotient/remainder pair are stored in AL/AH, AX/DX, or EAX/EDX
  * __STOS(B/W/D)__: writes the value AL/AX/EAX to EDI. Commonly used to initialize a buffer to a constant value
  * __SCAS(B/W/D)__: compares AL/AX/EAX with data starting at the memory address EDI
  * __LODS(B/W/D)__: reads 1, 2, or 4 byte value from esi and stores it in al, ax, or eax 
  * __REP__: repeats an instruction up to ECX times
  * __MOVS(B/W/D)__: moves data with 1, 2, or 4 byte granularity between two addresses. They implicitly use EDI/ESI as the destination/source address, respectively. In addition, they also automatically update the source/destination address depending on the direction flag
  * __CLD__: clear direction flag. DF: 0
  * __STD__: set direction flag. DF: 1. If DF is 1, addresses are decremented
  * __PUSHAD, POPAD__: pushes/pops all 8 general-purpose registers 
  * __PUSHFD, POPFD__: pushes/pops EFLAGS register 
  * __MOVSX__: moves a signed value into a register and sign-extends it 
  * __MOVZX__: moves an unsigned value into a register and zero-extends it
  * __CMOVcc__: conditional execution on the move operation. If the condition code's (cc) corresponding flag is set in EFLAGS, the mov instruction will be performed. Otherwises, it's just like a NOP instruction 
#
## *<p align='center'> x86-64 </p>*
* __Canonical Form__: all addresses and pointers are 64-bit, but virtual addresses must be in canonical form. Canonical form means that bit 47 and bits 48-63 must match since modern processors only support 48-bit for address space rather than the full 64-bit that is available. If the address is not in canonical form, an exception will be raised 
* __Registers__: aside from the old x86 registers being extended from 32-bit to 64-bit, extra General Purpose Registers have also been added (e.g. r8 to r15)
  * 16 general-purpose registers each 64-bits (RAX, RCX, RDX, RBX, RSP, RBP, RSI, RDI, R8, R9, R10, R11, R12, R13, R14, R15)
    + DWORD (32-bit) version can be accessed with a D suffix, WORD (16-bit) with a W suffix, BYTE (8-bit) with a B suffix for registers R8 to R15
    + For registers with alternate names like x86 (e.g. RAX, RCX), size access for register is same as x86. For example, 32-bit version of RAX is EAX and the 16-bit version is DX 
    * RBP is treated like another GPR. As a result, local variables are referenced through RSP
  * 16 XMM registers, each 128-bit long. XMM registers are for SIMD instruction set, which is an extension to the x86-64 architecture. SIMD is for performing the same instruction on multiple data at once and/or for floating point operations 
    + Floating point operations were once done using stack-based instruction set that accesses the FPU Register Stack. But now, it can be done using SIMD instruction set 
* __Calling Conventions__: Parameters are passed to registers instead of the stack. Although additional ones will still be stored on stack
  + __Windows__: first 4 parameters are placed in RCX, RDX, R8, and R9
  + __Linux__: first 6 parameters are placed in RDI, RSI, RDX, RCX, R8, and R9
* __Exception Handling__:  
  * __Windows__: on x86, Structured Exception Handling (SEH) is stored on the stack, which makes it vulnerable to buffer overflow attacks. SEH on x64 is implemented differently. SEH is table-based and no longer stored on the stack. It is instead stored in the PE Header
  * __Linux__: for both x86 and x64, exception handling can be achieved through signals 
* __Other Notable Differences From x86__: 
  * Non-leaf functions are sometimes called frame functions because they require a stack frame. Since stack space on x86-64 cannot change in the middle of a function,  all nonleaf functions are required to allocate 0x20 bytes of stack space when they call a function. This allows the function being called to save the register parameters (RCX, RDX, R8, and R9) in that space. If a function has any local stack variables, it will allocate space for them in addition to the 0x20 bytes
  * Easier in 64-bit code to differentiate between pointers and data values. The most common size for storing integers is 32 bits and pointers are always 64 bits
  * In 32-bit code, stack space can be allocated and unallocated in middle of the function using push or pop. However, in 64-bit code, functions cannot allocate any space in the middle of the function
  * Supports instruction pointer-relative addressing on data. Unlike x86, referencing data will not use absolute address but rather an offset from RIP
#
## *<p align='center'> ARM </p>*
* __ARM Version__:
  * ARMv7 uses 3 profiles (Application, Real-Time, Microcontroller) and model name (Cortex). For example, ARMv7 Cortex-M is meant for microcontroller and support Thumb-2 execution only 
  * Thumb-1 is used in ARMv6 and earlier. Its instructions are always 2 bytes in size
  * Thumb-2 is used in ARMv7. Its instructions can be either 2 bytes or 4 bytes in size. 4 bytes Thumb instruction has a .W suffix, otherwise it generates a 2 byte Thumb instruction
  * Native ARM instructions are always 4 bytes in size
* __Privileges Separation__ are defined by 8 modes. In comparison to x86, User (USR) mode is like ring3 and Supervisor (SVC) mode is like ring0
* __Registers__: there are 16 32-bit general-purpose registers (R0 - R15), but only the first 12 registers are for general purpose usage
    + R0 holds the return value from function call
    + R13 is the stack pointer (SP)
    + R14 is the link register (LR), which holds return address for function call
    + R15 is the program counter (PC)
  * Control register is the current program status register (CPSR), also known as application program status register (APSR), which is basically an extended EFLAGS register in x86
* __Load/Store Instructions__: only load/store instructions can access memory. All other instructions operate on registers 
  + load/store instructions: LDR/STR, LDM/STM, and PUSH/POP
  #
  __LDR/STR__
  * There are 4 forms of LDR/STR instructions 
    + LDR/STR Ra, [Rb]. LDR loads the data at Rb to Ra. STR stores Ra to the location pointed to by Rb 
    + LDR/STR Ra, [Rb, imm]
    + LDR/STR Ra, [Rb, Rc]
    + LDR/STR Ra, [Rb, Rc, barrel-shifter]. Barrel shifter is performed on Rc. The result is added to Rb, the base address 
    + extra (pseudo-form): LDR Ra, ="address". This is not valid syntax, but is used by disassembler to make disassembly easier to read. Internally, what's actually executed is LDR Ra, [PC + imm]
  * There are 3 addressing modes for LDR/STR: offset, pre-indexed, post-indexed 
    + __Offset__: base register is never modified 
      * Form: LDR Rd, [Rn, offset]
    + __Pre-indexed__: base register is updated with the memory address used in the reference operation 
      * Form: LDR Rd, [Rn, offset]!
    + __Post-indexed__: base register is used as the address to reference from and then updated with the offset 
      * Form: LDR Rd, [Rn], offset
  #
  __LDM/STM__
  * LDM/STM loads/stores multiple words (32-bits), starting from a base address 
    * LDM Form: LDM<-mode-> Rb [!], {Registers}. LDM means load the data starting from Rb and loads it into Registers
    * STM Form: STM<-mode-> Rb [!], {Registers}. STM means store the data in Registers into location starting from Rb
      * Exclamation mark (!) means Rb should be updated to the final location after the LDM/STM operation
  * LDM/STM can use several types of stack: 
    + Descending or ascending: descending means that the stack grows downward, from higher address to lower address. Ascending means that the stack grows upward 
    + Full or empty: full means that the stack pointer points to the last item in the stack. Empty means that the stack pointer points to the next free space
    + Full descending: STMFD (STMDB), LDMFD (LDMIA)
    + Full ascending: STMFA (STMIB), LDMFA (LDMDA)
    + Empty descending: STMED (STMDA), LDMED (LDMIB)
    + Empty ascending: STMEA (STMIA), LDMEA (LDMDB)
  #
  __PUSH/POP__
  * PUSH/POP's form: PUSH/POP {register(s)}
  * PUSH/POP and STMFD/LDMFD are functionally the same, but PUSH/POP is used as prologue and epilogue in Thumb state while STMFD/LDMFD is used as prologue and epilogue in ARM state. 
* __Instructions For Function Invocation__: B, BX, BL, and BLX
  + B's syntax: B imm. imm is relative offset from R15, the program counter
  + BX's syntax: BX &lt;register&gt;. X means that it can switch between ARM and THUMB state. If the LSB of the destination is 1, it will execute in Thumb state. BX LR is commonly used to return from function 
  + BL's syntax: BL imm. It stores return address, the next instruction, in LR before transferring control to destination. To return from function, B LR is used
  + BLX's syntax: BLX imm./&lt;register&gt;. When BLX uses an offset, it always swap state (ARM to THUMB or THUMB to ARM)
* __Conditional Execution__: instructions can be conditionally executed by adding conditional suffixes
  * This is how conditional branch instruction is implemented. For example, BEQ executes Branch instruction if EQ condition is true (ZF = 1)
  * Thumb instruction cannot be conditionally executed, with the exception of B instruction, without the IT instruction. 
    + IT (If-then)'s syntax: ITxyz cc. cc is the conditional suffix for the 1st instruction after IT. xyz are for the 2nd, 3rd, and 4th instructions after IT. It can be either T or E. T means that the condition must match cc for it to be executed. E means that condition must be the opposite of cc for it to be executed
  * For arithmetic operations, the "S" suffix will update conditional flags. Whereas, comparison instructions (CBZ, CMP, TST, CMN, and TEQ) automatically update the flags
* __Importance of Barrel Shifter__: since instructions can only be 2 or 4 bytes in size, it's not possible to directly use a 32-bit constant as an operand. As a result, barrel shifter can be used to transform the immediate into a larger value 
---

# .languages

## *<p align='center'> C++ Reversing </p>*
* __thiscall__: C++'s calling convention
  + On Microsoft Visual C++ compiled binary, this pointer is stored in ecx. Sometimes esi 
  + On g++ compiled binary, this pointer is passed in as the first parameter of the member function as an address 
  + Class member functions are called with the usual function parameters in the stack and with ecx pointing to the class’s object 
* __How An Object Is Represented__: Class’s object in assembly only contains the vfptr (pointer to virtual functions table) and variables. Member functions are not part of it
  + Child class automatically has all virtual functions and data from parent class
  + Even if the programmer did not explicit write the constructor for the class. If the class contains virtual functions, a call to the constructor will be made to fill in the vfptr to point to vtable. If the class inherit from another class, within the constructor there will have a call to the constructor of the parent class  
  + vtable of a class is only referenced directly within the class constructor and destructor
  + Compiler places a pointer immediately prior to the class vtable. It points to a structure that contains information on the name of class that owns the vtable
* __Name Mangling__ is used to support Method Overloading (multiple functions with same name but accept different parameters). Since in PE or ELF format, a function is only labeled with its name 
#
## *<p align='center'> Python Reversing </p>*
* __PVM (Python Virtual Machine)__ is a stack-based virtual machine that stores operands in an upwardly-growing stack
  * In stack-based execution, an operation is performed by popping operands from the stack, operates on them, and then storing the result back to the stack
  * __Advantages of Stack-Based Virtual Machine__
    * Instructions are shorter than instructions in register-based virtual machine since operands don’t need to be explicitly stated because they are on the stack, thus resulting in shorter overall bytecode 
    * Easier to implement since it doesn’t have to worry about register allocation 
* __The 3 Tuples associated with Function Object__
  * Local variables and parameters are stored in &lt;object name&gt;.`__code__`.co_varnames
  * Global variables that it uses are stored in &lt;object name&gt;.`__code__`.co_names
  * Constants it uses are stored in &lt;object name&gt;.`__code__`.co_consts
* __Python Bytecode Instructions__ 
  * For full instruction set, check out the [Official Python Documentation](https://docs.python.org/2/library/dis.html)
  * An instruction consists of an opcode and possibly an oparg. Opcode is the instruction and oparg is the index that resolves to the actual operands
    * Some instructions, such as BINARY_ADD, doesn't require an oparg since the operands it needs are on the stack 
    * Other instructions, such as those for LOAD/STORE, require an oparg to index a tuple for the operand. The specific tuple is referenced in the latter half of the opcode
      * __LOAD/STORE_CONST__: uses oparg to index the &lt;object name&gt;.`__code__`.co_consts tuple 
      * __LOAD/STORE_FAST__: uses oparg to index the &lt;object name&gt;.`__code__`.co_varnames tuple
      * __LOAD/STORE_GLOBAL__: uses oparg to index the &lt;object name&gt;.`__code__`.co_names tuple
      * Load will push the value it indexed onto the stack and store will store the value at the top of the stack into the indexed object 
---

# .file-formats

## *<p align='center'> ELF Files </p>*
<p align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/file-formats/ELF_Files/elf_file_format.png" height="400"> 
</p>

* What's important for linking compared to what's important for execution: 

<p align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/file-formats/ELF_Files/loading_elf_file.png" height="400"> 
</p>

* __ELF File Header__ starts at offset 0 and is the roadmap that describes the rest of the file. It marks the ELF type, architecture, execution entry point, and offsets to program headers and section headers
* __Program Header Table__ let the system knows how to create the process image. It contains an array of structures, each describing a segment. A segment contains one or more sections
* __Section Header Table__ is not necessary for program execution. It is mainly for linking and debugging purposes. It is an array of __ELF_32Shdr__ or __ELF_64Shdr__ structures (Section Header)
  * __.got (Global Offset Table) Section__: a table of addresses located in the data section. It allows PIC (Position Independent) code to reference data that were not available during compilation (ex: extern "var"). That data will have a section in .got, which will then be filled in later by the dynamic linker
  * __.plt (Procedure Linkage Table) Section__: contains within the text segment, consisting of external function entries. Each plt entry has a correcponding entry in .got.plt which contains the actual offset to the function
    * plt entry consists of: 
      + A jump to an address specified in GOT
      + argument to tell the resolver which function to resolve (only reach there during function's first invocation)
      + call the resolver (resides at PLT entry 0)
  * __.got.plt Section__: contains dynamically-linked function entries that can be resolved lazily through lazy binding. This means that it doesn't resolve the address until the function is called
* __Useful Compilation Options To Know For GCC__:
  * __-g__: the compiled binary will contain extra sections with names that start with .debug_. The most important one of the .debug section is .debug_info. It tells you the path of the source file, path of the compilation directory, version of C used, and the line numbers where variables are declared in source code. It will also contain the parameter names for local functions
  * __-s__: the compiled binary will not contain symbol table and relocation information. This means that the .symtab will be stripped away, which contains references to variable and local function names
  * __-O3__: the second highest optimization level. The optimizations that it applied will actually result in bigger overall file size than the compiled version of the unoptimized binary
  * __-funroll-loops__: unroll the looping structure of any loops, making it harder for reverse engineer to analyze the compiled binary
* __Stripped Binary__: there are 2 sections that contain symbols: .dynsym and .symtab. .dynsym contains dynamic/global symbols, those symbols are resolved at runtime. .symtab contains all the symbols
  * __nm command__ to list all symbols in the binary from .symtab
  * Stripped binary == no .symtab symbol table
  * .dynsym symbol table cannot be stripped since it is needed for runtime, so imported library functions' symbols remain in a stripped binary. But if a binary is compiled only with statically-linked libraries, it will contain no symbol table at all if stripped
  * With non-stripped, gdb can identify local function names and knows the bounds of all functions so we can do this: disas &lt;function name&gt;
  * With stripped binary, gdb can’t even identify main. We can still try to identify main from the entry point using the command: __info file__. Also, can’t use disas since gdb does not know the bounds of a functions so it does not know which address range should be disassembled. Solution: use examine(x) command on address pointed by pc register like: __x/14i $pc__
* __Useful Tools To Analyze ELF File__: 
  + display section headers: __readelf -S &lt;file&gt;__
  + display program headers and section to segment mapping: __readelf -l &lt;file&gt;__
  + display symbol tables: __readelf --syms &lt;file&gt;__ or __objdump -t &lt;file&gt;__
  + display a section's content: __objdump -s -j &lt;section name&gt; &lt;file&gt;__
  + trace library call: __ltrace -f &lt;file&gt;__
  + trace sys call: __strace -f &lt;file&gt;__
  + decompile: check out __https://retdec.com/__
  + view a running program's process address space: __/proc/$pid/maps__
#
## *<p align='center'> PE Files </p>*

<p align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/file-formats/PE_Files/pe_header.png"> 
</p>

* How a PE file is loaded into memory: 

<p align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/file-formats/PE_Files/loading_pe_file.png"> 
</p>

* __Virtual Address(VA) to File Offset Translation__: file_offset = VA - image_base - section_base_RVA + section_file_offset
  1. VA - image_base = RVA. RVA (Relative Virtual Address) is virtual address relative to the image base (HMODULE). It is used to avoid hardcoded memory addresses since the image base might not always get loaded to its preferred load address. As a result, address obtained from disassembler might not match the address obtained from a debugger
  2. RVA - section_base_RVA = offset from base of the section
  3. offset_from_section_base + section_file_offset = file offset on disk  
* __DOS Header__: starts at offset 0. The 2 fields of interest in the DOS header are __e_magic__ and __e_lfanew__. __e_magic__ contains the magic number 0x5A4D (MZ). __e_lfanew__ contains PE header's file offset
  * __e_lfanew__ field is necessary since between DOS Header and PE Header is the DOS Stub, which for backward compatibility prints "This program cannot be run in DOS mode" if a 32-bit PE file is ran in a 16-bit DOS environment
* __PE Header (IMAGE_NT_HEADERS)__ contains 3 fields
  * __Signature__: always 0x00004550 ("PE00")
  * __IMAGE_FILE_HEADER (COFF Header)__: contains basic information on the file (e.g. target CPU, number of sections)
  * __IMAGE_OPTIONAL_HEADER32 (PE Optional Header)__: contains most of the meaningful information. Can be further broken down into 3 parts:
    * __Standard Fields__: notable fields are MagicNumber and AddressOfEntryPoint
      * __MagicNumber__: determines whether the image file uses 64-bit address space (PE32+) or 32-bit address space. PE32+ contained widened PE Optional Header
      * __AddressOfEntryPoint__: RVA to the entry point function
    * __Windows Specific Fields (Additional Fields)__: notable fields are ImageBase, SectionAlignment, and SizeOfImage
      * __ImageBase__: preferred address when loaded into memory
      * __SectionAlignment__: alignment for section when loaded into memory such that a section's VA will always be a multiple of this value 
      * __SizeOfImage__: size of image starting from image base to the last section rounded to the nearest multiple of SectionAlignment
    * __Data Directory (IMAGE_DATA_DIRECTORY)__: array of IMAGE_DATA_DIRECTORY structures that contains RVAs and sizes of many important data structures (e.g. imports, exports, base relocations). Locations of those data structures reside in sections. Data structures allows loader to quickly locate an important section. Here are the notable IMAGE_DATA_DIRECTORY entries:
      * __Program Exception Data__: pointed by IMAGE_DATA_DIRECTORY's entry IMAGE_DIRECTORY_ENTRY_EXCEPTION. It is an exception table that contains an array of IMAGE_RUNTIME_FUNCTION_ENTRY structures. Each IMAGE_RUNTIME_FUNCTION_ENTRY contains the address to an exception handler
      * __Base Relocations__: pointed by IMAGE_DATA_DIRECTORY's entry IMAGE_DIRECTORY_ENTRY_BASERELOC. Refers to as the .reloc section. Contains every location that needs to be rebased if the executable doesn't load at the preferred load address
* __Section Header Table (IMAGE_SECTION_HEADERs)__ is right after PE Header. Each IMAGE_SECTION_HEADER contains information on a section, such as a section's virtual address or its pointer to data on disk 
  * 2 sections can be merged into a single one if they have similar attributes
    * .idata is often merged into .rdata in release-mode executable 
* __Overlay__: data appended to end of a PE File. It is not loaded into memory
---

# .anti-analysis

## *<p align='center'> Obfuscation </p>*
* __Definition__: Program transformation techniques that output a program that is semantically equivalent (execute the same) to the original program but is more difficult to analyze
* __Original Entry Point (OEP) Hiding__: OEP can be hidden through packing. A packer can compress or encrypt a whole executable and inject an unpacking stub that unpack (decompress or decrypt) the executable during runtime. This will hide the OEP and also the original executable (such as the text, data, rsrc sections) from static analysis
  * __Tail Jump__: an instruction that jumps from the unpacking stub to OEP after the unpacking stub finishes
  * __Signs Of Packer Usage__: 
    * Unusually high file entropy 
    * Program behavior that mimics the way system loader loads a executable file into memory
  * __How To Hide From Detection__: 
    * Instead of encrypting/packing the whole binary, only encrypt/pack a small section of it (e.g. .text section) to avoid high file entropy
* __Functions In/Out-Lining__: performs operations that create inline and outline functions randomly and through multiple passes to obfuscate the call graph  
  * __Inline Functions__: a function that is merged into the body of its caller 
  * __Outline Functions__: inverse of Inline function where a subportion of a function is extracted to create another function. The subportion of this function is then replaced with a CALL instruction that calls the new function 
* __Opcode Obfuscation__: an effective technique for preventing correct disassembly by displaying opcodes that need to be re-interpreted/altered during runtime before being executed by the CPU 
  + __Self-Modifying Code__: disassembly shows the encoded/encrypted opcodes. During runtime, the encoded/encrypted opcodes are decoded/decrypted, revealing the opcodes that will actually be executed. Encoding/encrypting portions of a program can hinder static analysis because disassembly is not possible (if it is, it will show false disassembly instead) and hinder debugging because placing breakpoints is difficult. For example, even if the start of an instructions is known, breakpoint cannot be placed until the instruction have been decoded/decrypted
  + __Virtual Obfuscation (Runtime Code Generation)__: parts of the program are compiled to the bytecode that corresponds to the instruction set of an undocumented interpreter (usually one that the obfuscator wrote him or herself). The interpreter will be a part of the protected program such that during runtime, the interpreter will translate those bytecode into machine code that corresponds to the original architecture (e.g. x86). This method can also prevent the dumping of a whole section of de-obfuscated opcodes from memory during runtime like the Self-Modifying Code method would
* __Dynamically Computed Target Addresses__: an address to which execution will go to is computed at runtime. This makes it hard to tell what the call destination is statically
* __Dead Code Insertion__: inserts useless code that doesn't affect a program's functionalities. To do this, an obfuscator needs to know which registers are dead (e.g. does not contain important values) so that it can insert instructions that modify those dead registers 
  * Can be resolved by applying dead code elimination
* __Junk Code Insertion__: inserts code that never get executed 
* __Pattern-Based Obfuscation__: transforms a sequence of instructions into another sequence of instructions that is more complicated but semantically the same
  * This task could be difficult since the obfuscated sequence can look semantically the same but does not preserve CPU state, which can cause later program behavior to be semantically nonequivalent. This can happen if the new sequence's side effects (modify/didn't modify the flags or modify/didn't modify the stack) is opposite that of the original sequence's side effects  
  * [Reverser's Living Nightmare](https://0x00sec.org/t/what-a-living-nightmare-looks-like/2087)
  * Can be resolved by applying peephole optimization
* __Destruction of Sequential and Temporal Locality (Spaghetti Code)__: code within a basic block will be right next to each other __(sequential locality)__ and basic blocks relating to each other will be placed in close proximity to maximize instruction cache locality __(temporal locality)__. To obstruct this property and make disassembly harder to understand, a basic block can be further divided and randomized using unconditional jumps
* __Opaque Predicate__: transforms a trivial opaque predicate into a non-trivial opaque predicate. On the source code level, a trivial opaque predicate's conditional construct will be optimized away if compiler knows that it will always be evaluated to either True or False. For a non-trivial opaque predicate, even though the predicate will always evaluate to either True or False, the underlying code construct makes it hard to figure that out statically. As a result, compiler doesn't optimize away the conditional construct
  * [Environment-Based Opaque Predicates](https://reverseengineering.stackexchange.com/questions/2340/how-to-design-opaque-predicates)
  * Uses global variables instead of constants in the predicates. Compiler won't be able to optimize the conditional construct since it can't assume the value of global variables 
  * Introduces entropy into the predicate (e.g. using rand() function)
* __Control-Flow Graph Flattening__: obfuscates control flow by replacing a group of control structures with a dispatcher. Each basic block updates the dispatcher's context so it knows which basic block to execute next
* __Irreducible Programs__: transform a loop into more complicated construct on the source code level. Duplicate the loop and insert a conditional construct that could reach either loop. Obsfucate each loop accordingly. When compiled, compiler cannot optimize away the conditional construct since even though both path are functionally the same they are syntactically different, so both obfuscated loops remain in compiled binary
* __Constant Unfolding__: replaces constant with unnecessary computations that will output the same constant
  * this can be accomplished on the source code level if the variable is assigned as volatile in C. The volatile keyword tells the compiler to not optimize this variable so you can perform unnecessary computations on it without worrying that the compiler will optimize those computations away 
  * Can be resolved by applying constant folding optimization on the obfuscated sequence of instructions that performs constant unfolding
* __Arithmetic Substitution via Identities__: replaces a mathematical statement with one that is more complicated but semantically the same
* __Array-Based Obfuscation__: alters the data structures used in the program to make it harder to analyze 
  * __Aggregation__: splits, merges, folds, or flattens an array. [This technique can be used to hide the intended string](https://youtu.be/7IKIzsXr3Z8?t=1964) or simply makes an array harder to analyze by merging junk array with the actual array
  * __Re-Ordering__: obfuscation technique that re-orders the array. The indices used to access elements in the array now needs to be updated and can be made more complicated by using a function that maps indice i to the new indice
  * __Encoding__: alters how the program interprets stored data. For example, initialize an index i by 2i instead. When the index i needs to be used, use the expression i/2 instead. It is also commonly used to hide readable strings by only storing the encoded strings in the binary and decode those strings during runtime 
  * __Table Interpretation__: loads an array with call destinations and junk data. Instead of directly calling those functions, call them indirectly by indexing the array  
* __Imported Function Obfuscation (makes it difficult to determine which shared libraries or library functions are used)__: have the program’s import table be initialized by the program itself. The program itself loads any additional libraries it depends on, and once the libraries are loaded, the program locates any required functions within those libraries
  + (Windows) use LoadLibrary function to load required libraries by name and then perform function address lookups within each library using GetProcAddress
  + (Linux) use dlopen function to load the dynamic shared object and use dlsym function to find the address of a specific function within the shared object 
#
## *<p align='center'> Anti-Disassembly </p>*
* __Disassembly Technique__: ways to disassemble machine code
  * __Linear Disassembly__: disassembling one instruction at a time linearly. Problem: code section of nearly all binaries will also contain data that isn’t instructions 
  * __Flow-Oriented Disassembly__: for conditional branch, it will process false branch first and note to disassemble true branch later. For unconditional branch, it will add destination to the end of list of places to disassemble in future and then disassemble from that list. For call instruction, most will disassemble the bytes after the call first and then the called location. If there is conflict between the true and false branch when disassembling, disassembler will trust the one it disassembles first
* __Disassembly Desynchronization__: to cause disassembly tools to produce an incorrect program listing. Works by taking advantage of the assumptions made by flow-oriented disassemblers. For every assumption it makes (e.g. process false branch first), there is a corresponding anti-disassembly technique. Desynchronization has the greatest impact on the disassembly, but it is easily defeated by reformatting the disassembly to reflect the correct instruction flow
  * __Opaque Predicate__: conditional construct that looks like conditional code but actually always evaluates to either true or false 
    + __Jump Instructions With The Same Target__: JZ follows by JNZ. Essentially an unconditional jump. The bytes following JNZ instruction could be data but will be disassembled as code
    + __Jump Instructions With A Constant Condition__: XOR follows by JZ. It will always jump so bytes following false branch could be data but will be disassembled as code
  + __Impossible Disassembly__: a byte is part of multiple instructions. Disassembler cannot represent a byte as part of two instructions. Either can the processor, but it doesn't have to because it just needs to execute the instructions 
* __Function Pointer Problem__: if a function is called indirectly through pointers (on the stack), IDA xref will only record the first usage (i.e. when the offset is loaded onto the stack). As a result, xref will not show the various locations that the function could have been called from
* __Processer-Based Control Indirection__: 
  * __CALL Instruction Abuse__: Using CALL instruction to jump to new location instead of a function's entry point. The address pushed onto the stack by CALL can be discarded with the instruction ADD ESP, 4. This will cause false disassembly since the disassembler will try label the CALL location as a function's entry point when it's not
  * __Return Pointer Abuse__: RET is used to jump to function or other location instead of returning from function using the PUSH, RET sequence. Disassembler won’t show any code cross-reference to the target being jumped to. Also, disassembler will prematurely terminate the function since RET is supposed to be used for returning from function, resulting in false disassembly
* __Thwarting Stack-Frame Analysis__: technique to mess with IDA when deducing numbers of parameters and local variables. For example, the code makes a conditional jump that's always false but in true branch add absurd amount to ESP. If the disassembler chooses to believe the true branch, the numbers of local variables will be incorrect
* __Parser Differential Attack (File Format Hacks)__: makes modifications to the ELF file such that it will still execute fine, but the disassembler/debugger will not work properly if you loads a binary into it. Similar techniques can be done on other file format like Portable Executable (PE) as well
  * __Tampering/Removing Section Headers (Linux)__: makes tools such as gdb and objdump useless since they rely on the section headers to locate information regarding the various sections. Segments are necessary for program execution, not sections. Section headers table is for linking and debugging
    + "The biggest fuck up that (most) analysis tools are doing is that they are using the section headers table of the ELF file to get information about the binary...[because] sections contain more detailed information" - Andre Pawlowski
    + Modifying section headers' flag fields will make disassembler like IDA Pro to display incorrect disassembly listings. For example, changing .text section's flag from AX (alloc and execute) to WA (write and alloc), even though it still maps to the LOAD segment with flags RE (read and execute), will trick IDA into not disassembling main along with other local functions. It's because the section headers table tells IDA that the area those functions reside in is not executable
    + In addition to changing the section headers' flags, if we include fake .init section we can trick IDA into not disassembling any of the code starting from the entry point. This can happen since IDA will try to disassemble the .init section before the entry point. But if the .init section overlaps with the entry point, then entry point will not get disassembled at all, especially when the entry point is not marked as executable in the section headers (by messing with the section flag)
    + The entry point provided in the ELF Header is in virtual address. To get the actual offset, find the section that the entry point's virtual address fall under. Once you identified the section, use the section header to figure out entry point's offset: (e_entry - sh_addr) + sh_offset. So if you alter sh_addr (virtual address) field of the section that contains the entry point, disassembler that relys too much on the section headers table (e.g. IDA) won't be able to find the correct entry point in the file. This technique won't work on Radare2 since it prefers information in program headers even if the section headers exist 
    + __Mixing Symbols__: appends a fake dynamic string table to the end of the binary and overwrite offset of .dynstr entry in section headers table with offset of the fake dynamic string table. This will make imported functions display the fake symbol names
    + If you remove the section headers table, disassembler/debugger will have to rely on program headers even though program headers give us less information. For example, .text, .rodata, and dynsym all belong to the same segment. And without section headers table, we won't be able to differentiate between the sections within a segment. But fully relying on program headers can also lead to failure. For example, another technique to make IDA fail to load an ELF file is to find a program header that is not required for loading and change the offset field to point to a location that is outside the binary
  * __ELF Header Modification__: inserting false information into ELF Header to discourage analysis
    + Simply zero-ing out information regarding section headers table in the ELF Header (e_shoff, e_shentsize, e_shnum, e_shstrndx) can make tools such as readelf and Radare2 unable to display sections even though Section Headers Table still exists within the binary
    + The 6th byte of the ELF Header is EI_DATA, residing within e_ident array, which makes up the first 16 bytes of the ELF Header. EI_DATA specifies the data encoding of the processor-specific data in the file (unknown, little-endian, big-endian). Modifying EI_DATA after compilation will not affect program execution, but will make tools such as readelf, gdb, and radare2 to not work properly since they use this value to interpret the binary
#
## *<p align='center'> Anti-Debugging </p>*
* __Using Functions from Dynamically Linked Libraries to Detect Debugger's Presence__ 
  * __ptrace (Linux)__: The ptrace system call allows a process (tracer) to observe and control execution of a second process (tracee), but only one tracer can control a tracee at a time. All debuggers and program tracers use ptrace call to setup debugging for a process. If the debugee's code itself contains a ptrace call with the request type PTRACE_TRACEME, PTRACE_TRACEME will set the parent process (most likely bash) as the tracer. This means that if a debugger is already attached to the debugee, the ptrace call within the debugee's code will fail 
    + This method can be bypassed by using LD_PRELOAD, which is an environment variable that is set to the path of a shared object. That shared object will be loaded first. As a result, if that shared object contains your own implementation of ptrace, then your own implementation of ptrace will be called instead when the call to ptrace is encountered 
      * [How To Deter People From Using LD_PRELOAD To Bypass Ptrace](https://seblau.github.io/posts/linux-anti-debugging)
  * Windows API provides several functions that can be used by a program to determine if it is being debugged
    * __DebugActiveProcess (Self-Debugging)__: Windows' version of ptrace. Self-debugging can be achieved by spawning a new process and use kernel32!DebugActiveProcess to debug the parent process. Call to DebugActiveProcess will fail if a debugger is already attached to the parent process    
    * __IsDebuggerPresent__: checks the PEB structure for the IsDebugged field. If the process is running under the context of a debugger, the IsDebugged field will be 1, otherwise 0
    * __CheckRemoteDebuggerPresent__: can use this to check if a remote or current process is being debugged. Also checks the IsDebugged field in the PEB structure to make the decision
    * __NtQueryInformationProcess__: Native Windows API in Ntdll.dll. CheckRemoteDebuggerPresent will eventually call this native function. Passing ProcessDebugPort (0x7) as its second argument will tell this function to query whether the process is being debugged or not 
    * __OutputDebugString__: sends debugger strings to display. If debugger is present, this function will return a valid address in the process address space into eax. Otherwise, it will return an invalid address 
    * For a more comprehensive list, check out [section 7 of the Ultimate Anti-Debugging Reference by Peter Ferrie](http://anti-reversing.com/Downloads/Anti-Reversing/The_Ultimate_Anti-Reversing_Reference.pdf)
  * __readlink (Linux)__: calling readlink on "/proc/&lt;ppid&gt;/exe" will return a string containing the location of the debugger if one is attached. You can find ppid by checking the dynamic file /proc/&lt;pid&gt;/status. And to find the pid of the process, use the ps command
    * __/proc/&lt;pid&gt;/status__: this dynamic file also contains other information on a running process, such as whether or not the process is being traced. If the field tracerPid is 0, the process is not being traced
    * Under GDB, argv[0] (name of the current invoked program) contains binary's absolute path even if you invoke the binary from a relative path. Under normal execution, argv[0] will contain the relative path. You can take advantage of this with any string-related functions, such as strcmp() and strstr(), to detect presence of GDB

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/anti-analysis/Anti-Debugging/proc_status.png"> 
<p align='center'><sub><strong>a tracee is identified using /proc/&lt;pid&gt;/status and its corresponding tracer is identified using readlink</strong></sub></p>
</div>

* __Using Flags within the PEB structure to Detect Debugger's Presence (Windows)__
  * Location of PEB can be referenced by the location fs:[30h]. The second item on the PEB struct is BYTE BeingDebugged. The API function, isDebuggerPresent, checks this field to determine if a debugger is present or not
  * __Flags and ForceFlags__: within Reserved4 array in PEB, is ProcessHeap, which is set to location of process’s first heap allocated by loader. This first heap contains a header with fields that tell kernel whether the heap was created within a debugger. The fields are Flags and ForceFlags. If the Flags field does not have the HEAP_GROWABLE(0x2) flag set, then the process is being debugged. Also, if ForceFlags != 0, then the process is being debugged. The location of both Flags and ForceFlags in the heap depends on whether the machine is 32-bit or 64-bit and also the version of Window Operating System (e.g. Windows XP, Windows Vista)
  * __NTGlobalFlag__: Since processes run slightly differently when started by a debugger, they create memory heaps differently. The information that the system uses to determine how to create heap structures is stored in the NTGlobalFlag field in the PEB at offset 0x68 in x86 and 0xbc in x64. If value at this location is 0x70 (FLG_HEAP_ENABLE_TAIL_CHECK(0x10) | FLG_HEAP_ENABLE_FREE_CHECK(0x20) | FLG_HEAP_VALIDATE_PARAMETERS(0x40)), we know that we are running in debugger
* __Breakpoint Detection__: if we detected breakpoints in a process, then we know that process is running under a debugger
  * __INT Scanning__: Search the .text section for the 0xCC byte. If it exists, that means that a software breakpoint has been set and the process is under a debugger 
  * __Code Checksums__:  Instead of scanning for 0xCC, this check simply performs a cyclic redundancy check (CRC) or a MD5 checksum of the opcodes. This not only catches software breakpoints, but also code patches 
  * __Anti-Step-Over__: the rep or movs(b|w|d) instruction can be used to overwrite/remove software breakpoints that are set after it
  * __Hardware Breakpoints (Windows)__: Get a handle to current thread using GetCurrentThread(). Get registers of current thread using GetThreadContext(). Check if registers DR0-DR3 is set, if it is then there are hardware breakpoints set. On Linux, user code can't access hardware breakpoints so it's not possible to check for it  
* __Interrupts__: Manually adding/setting interrupts to the code to help detect present of a debugger
  + __False Software Breakpoints__: a breakpoint is created by overwriting the first byte of instruction with an int3 opcode (0xcc). To setup a false breakpoint then we simply insert int3 into the code. This raises a SIGTRAP signal when int3 is executed. If our code has a signal handler for SIGTRAP, the handler will be executed before resuming to the instruction after int3. But if the code is under the debugger, the debugger will catch the SIGTRAP signal instead and might not pass the signal back to the program, resulting in the signal handler not being executed
  * __False Memory Breakpoints__: Create a dynamic buffer and write the opcode for ret instruction in it. Manually change the permission of the page the buffer is in to guard. Pushes a return address to the stack before jumping to that dynamic buffer. If execution transfer control to that return address, then the program knows that it's under the context of a debugger since STATUS_GUARD_PAGE_VIOLATION exception was absorbed by the debugger 
  + __Two Byte Interrupt 3__: instead of 0xCC, it's 0xCD 0x03. Can also be used as false breakpoint
  + __Interrupt 0x2C__: raises a debug assertion exception. This exception is consumed by WinDbg 
  + __Interrupt 0x2D__: issues an EXCEPTION_BREAKPOINT (0x80000003) exception if no debugger is attached. Also it might also led to a single-byte instruction being skipped depending on whether the debugger chooses the EIP register value or the exception address as the address from which to resume 
  + __Interrupt 0x41__: this interrupt cannot be executed succressfully in ring 3 because it has a DPL of zero. Executing this interrupt will result in an EXCEPTION_ACCESS_VIOLATION (0Xc0000005) exception. Some debugger will adjust its DPL to 3 so that the interrupt can be executed successfully in ring 3. This results in the exception handler to not be executed
  + __ICEBP (0xF1)__: generates a single step exception
  + __Trap Flag Check__: Trap Flag is part of the EFLAGS register. IF TF is 1, CPU will generate Single Step exception(int 0x01h) after executing an instruction. Trap Flag can be manually set to cause next instruction to raise an exception. If the process is running under the context of a debugger, the debugger will not pass the exception to the program so the exception handler will never be ran
  + __Stack Segment__: when you operate on SS (e.g. mov ss, pop ss), CPU will lock all interrupts until the end of the next instruction. Therefore, if you are single-stepping through it with a debugger, the debugger will not stop on the next instruction but the instruction after the next one. One way to detect debugger is for the next instruction after a write to SS to be pushfd. Since the debugger did not stop there, it will not clear the trap flag and pushfd will push the value of trap flag (plus rest of EFLAGS) onto the stack

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/anti-analysis/Anti-Debugging/sigtrap.png"> 
<p align='center'><sub><strong>bypassing False Software Breakpoints with gdb</strong></sub></p>
</div>

* __Timing Checks__:  record a timestamp, perform some operations, take another timestamp, and then compare the two timestamps. If there is a lag, you can assume the presence of a debugger
  * __rdtsc Instruction (0x0F31)__: this instruction returns the count of the number of ticks since the last system reboot as a 64-bit value placed into EDX:EAX. Simply execute this instruction twice and compare the difference between the two readings
* __Detection Before main()__: make it harder to find anti-debugging checks, anti-debugging checks can be placed in code that executes before main()
  * __TLS Callbacks (Windows)__: most debuggers start at the program’s entry point as defined by the PE header. TlsCallback is traditionally used to initialze thread-specific data before a thread runs, so TlsCallback is called before the entry point and therefore can execute secretly in a debugger
  * In C, function using the "constructor" attribute will execute before main()
#
## *<p align='center'> Anti-Emulation </p>*
* __Definition__: using emulation allows reverse engineer to bypass many anti-debugging techniques
* __Detection through Syscall__: invoke various uncommon syscalls and check if it contains expected value. Since there are OS features not properly implemented, it means that the process is running under emulation
* __CPU Inconsistencies Detection__: try executing privileged instructions in user mode. If it succeeded, then it is under emulation
  + WRMSR is a privileged instruction (Ring 0) that is used to write values to a MSR register. Values in MSR registers are very important. For example, the SYSCALL instruction invokes the system-call handler by loading RIP from IA32_LSTAR MSR. As a result, WRMSR instruction cannot be executed in user-mode  
* __Timing Delays__: execution under emulation will be slower than running under real CPU
* __Number of Cores__: the number of cores under emulation could be smaller than the number of cores on the host machine 
#
## *<p align='center'> Bonus </p>*
* __Quote To Remember__: "From an anti-reversing prespective, code doesn't have to be hard to reverse engineer....all we really need in the end of the day is we need the reverse engineer give up" - Chris Domas 
  * [Repsych: Psychological Warfare in Reverse Engineering](https://www.youtube.com/watch?v=HlUe0TUHOIc)
  * [REpsych's Github Repo](https://github.com/xoreaxeaxeax/REpsych)
---

# .encodings

## *<p align='center'> String Encoding </p>*
* __ASCII__: string encoding that maps a byte to an English character, a special character, or a number
  * [ASCII Table](http://www.asciitable.com/)
  * Out of the 128 characters defined in ASCII, only 95 of them are human-readable
  * ASCII used 7 bits only, but the extra bit is still not enough to encode all the other languages
* __Unicode__: Various encoding schemes were invented but none covered every languages until Unicode came along
  * [Unicode Character Table](https://unicode-table.com/en/#control-character)
  * Unicode is a large table mapping every character to a unique numbers (code point) 
  * First 256 code points maps 1:1 to ASCII  
  * Different UTF encodings (e.g. UTF-8, UTF-16) use different amount of bytes to encode those code points
* __Cause Of Garbled Text__: reading a byte sequence using the wrong encoding scheme
  
#
## *<p align='center'> Data Encoding </p>*
* __Definition__: all forms of content modification for the purpose of hiding intent
* __Caesar Cipher__: formed by shifting the letters of alphabet #’s characters to the left or right
* __Single-Byte XOR Encoding__: modifies each byte of plaintext by performing a logical XOR operation with a static byte value
  * __Identifying XOR Loop__: looks for a small loop that contains the XOR function (where it is xor-ing a register and a constant or a register with another register)
  * __Single-byte XOR's Weakness__: if there are many null bytes then key will be easy to figure out since XOR-ing nulls with the key reveals the key. 
  * __Solutions To Single-Byte XOR Encoding's Weakness__: 
    + Null-preserving single-byte XOR encoding: if plaintext is NULL or key itself, then it will not be encoded via XOR
    + Generates the keystream used to XOR the data using a pseudorandom number generator 
      * __Blum Blum Shub PRNG__: generic form: Value<sup>i+1</sup> = (Value<sup>i</sup> * Value<sup>i</sup>) % M. M is a constant  that is the product of 2 large primes and an initial V needs to be given. Actual key being xor-ed with the data is the lowest byte of current PRNG value
* __Other Simple Encoding Scheme__:
  + __ADD, SUB__
  + __ROL, ROR__: Instructions rotate the bits within a byte right or left
  + __Multibyte__: XOR key is multibyte
  + __Chained or Loopback__: Use content itself as part of the key
    * the original key is applied at one side of the plaintext and the encoded output character is used as the key for the next character
* __Data Encoding Example (Base64)__:
  + Encodes binary data into character set of 64 ASCII characters
  * Most common character set is MIME’s Base64, whose table consists of A-Z, a-z, and 0-9 for the first 62 values and + / for the last 2 values
  * Base64 operates every 3 bytes (24 bits). For every 6 bits, it indexes the table with 64 characters. The encoded value is the character that is indexed with the 6 bits 
  * One padding character may be presented at the end of the encoded string (typically =) since Base64 operates every 3 bytes
  * Easy to develop a custom substitution cipher using Base64 since the only item that needs to be changed is the indexing string table of 64 characters
  * Example:

<p align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/encodings/Data_Encoding/base64_conversion.png"> 
</p>

[Go to .table-of-contents](#table-of-contents)
