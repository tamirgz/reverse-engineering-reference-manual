# <p align='center'> Reverse Engineering Cheatsheet </p>

<p align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-journal/blob/master/images/heading/Introduction.PNG"> 
</p>

__NOTE__: Here is a collage of reverse engineering topics that I find interesting. Enjoy~ 

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
* [.operating-system-concepts](#operating-system-concepts)
  + [Windows OS](#-windows-os-)
  + [Interrupts](#-interrupts-)
* [.anti-analysis](#anti-analysis)
  + [Obfuscation](#-obfuscation-)
  + [Anti-Disassembly](#-anti-disassembly-)
  + [Anti-Debugging](#-anti-debugging-)
  + [Anti-Emulation](#-anti-emulation-)
  + [Anti-Dumping](#-anti-dumping-)
  + [Bonus](#-bonus-)
* [.encodings](#encodings)
  + [String Encoding](#-string-encoding-)
  + [Data Encoding](#-data-encoding-)
---

# .general-knowledge

## *<p align='center'> int 0x7374617274 </p>*
* A process is a container for execution. A thread is what the OS executes
* Any function that calls another function is a non-leaf function, and all other functions are leaf functions
* The entry point of a binary (start function) is not main. A program's startup code (how main is set up and called) depends on the compiler and the platform that the binary is compiled for
  * Even if no library is statically compiled into the binary, part of the .text section will contain code that has nothing to do with what the original developer(s) wrote
* To hide a string from GNU's strings utility, construct the string in code. So instead of the string being referenced from the .data section, it will be constructed in the .text section 
  * One way to do this is to initialize a string as an array of characters assigned to a local variable. This will result in code that moves each character onto the stack one at a time. To make the character harder to recognize, check out the Data Encoding section
* __Random Number Generator__: Randomness requires a source of entropy (seed), which is an unpredictable sequence of bits that can come from the OS observing its internal operations or ambient factors. Algorithms using OS's internal operations or ambient factors as seed are known as pseudorandom generators, because while their output isn't random, it still passes statistical tests of randomness
* __Software/Hardware/Memory Breakpoint__: 
  * __Software Breakpoint__: debugger reads and stores the first byte of instruction and then overwrites that first byte with 0xCC (INT3). When CPU hits the breakpoint (0xCC), OS kernel sends SIGTRAP signal to process, process execution is paused, and internal lookup occurs to flip the original byte back
  * __Hardware Breakpoint__: set in special registers called debug registers (DR0 through DR7)
    + Only DR0 - DR3 registers are reserved for breakpoint addresses
    + Before CPU attempts to execute an instruction, it first checks whether the address is currently enabled for a hardware breakpoint. If the address is stored in debug registers DR0–DR3 and the read, write, or execute conditions are met, an INT1 is fired and the process halts
    + Can check if someone sets a hardware breakpoint on Windows by using GetThreadContext() and checks if DR0-DR3 is set
  * __Memory Breakpoint__: changes the permissions on a region, or page, of memory
    + Guard page: Any access to a guard page results in a one-time exception, and then the page returns to its original status. Memory breakpoint changes permission of the page to guard
* __Endianness__: Intel x86 and x86-64 use little-endian format. It is a format where multi-bytes datatype such as integer has its least significant byte stored in the lower address of main memory. Due to Intel's endianness, here are points to keep in mind when reversing: 
  * Characters or user input, whether constructed on stack or already initialized and placed in data section, will be placed in memory in the order that it comes in since each character is a byte long, so endianness doesn't matter. But if you read out multi-characters at a time, such as 4 characters using DWORD[addr], CPU will think that the 4 bytes at addr are in little-endian and will then retrieve those bytes in reverse order 
  * Multi-bytes datatype, such as integer, will be stored in little-endian in memory. But when accessed from memory, it will be in its original form since CPU assumes little-endian
  * Value stored in RAM is in little-endian but when moved to a register it is in big-endian
---

# .tools

## *<p align='center'> IDA Tips </p>*
* __Import Address Table (IAT)__: shows you all the dynamically linked libraries' functions that the binary uses. IAT is important for a reverser to understand how the binary is interacting with the OS. To hide APIs call from displaying in the IAT, a programmer can dynamically resolve an API call
  + __How To Find Dynamically Resolved APIs__: get the binary's function trace (e.g. hybrid-analysis (Windows sandbox), ltrace). If any of the APIs in the function trace is not in the IAT, then that API is dynamically resolved. Once you find a dynamically resolved API, you can place a breakpoint on the API in IDA's debugger view (go to Module Windows, find the shared library the API is under, click on the library and another window will open showing all the available APIs, find the API that you are interested in, and place a breakpoint on it). Once execution breaks there, step back through the call stack to find where it's called in user code
  * If there're functions in the IAT that are not in the function trace, that is considered normal since the function trace might not hit every single execution path. Through smart fuzzing, function trace coverage can be improved
* When IDA loads a binary, it simulates a mapping of the binary in memory. The addresses shown in IDA are the virtual memory addresses and not the offsets of binary file on disk
* __To Show Advanced Toolbar__: View -> Toolbars -> Advanced Mode
* __To Save Memory Snapshot From Your Debugger Session__: Debugger -> Take Memory Snapshot -> All Segments
* __Useful Shortcuts__: 
  + u to undefine 
  + d to turn it to data 
  + c to turn it to code 
  + g to bring up the jump to address menu
  + n to rename
  + x to show cross-references
#
## *<p align='center'> GDB Tips </p>*
* __Changing Default Settings__: 
  * ASLR is turned off by default in GDB. To turn it on: set disable-randomization off
  * Default displays assembly in AT&T notation. To change it to the more readable and superior Intel notation: set disassembly-flavor intel. 
  * To make changes permanent, write it in the .gdbinit file
* __User Inputs__: way to pass user inputs to debugged program as arguments or/and as stdin
  * After already starting GDB...
    * (gdb) run &lt;argument 1&gt; &lt;argument 2&gt; __<__ &lt;file&gt;
    * content of file will be passed to debugged program's stdin
* __Automation__: ways to automate tasks in GDB
  * __-x Option__: puts the list of commands you want GDB to run when gdb starts in a file. Run GDB with the -x option like this:
    * gdb -x &lt;command file&gt; &lt;program to debug&gt;
  * __Hooks__: user-defined command. When command ? is ran, user-defined command 'hook-?' will be executed (if it exists)
    + When reversing, it could be useful to hook on breakpoints by using hook-stop 
    + How to define a hook: 
      1. __(gdb)__ define hook-?
      2. __>__ ...commands...
      3. __>__ end
      4. __(gdb)__
* maint info sections: shows where sections are mapped to in virtual address space
* i command displays information on the item specified to the right of it
  + __i proc mappings__: show mapped address spaces 
  + __i b__: show all breakpoints 
  + __i r__: show the values in registers at that point of execution
* x command displays memory contents at a given address in the specified format
  + Since disas command won't work on stripped binary, x command can come in handy to display instructions from current program counter: x/14i $pc
* p command displays value stored in a named variable
* Catch program events: catch &lt;event&gt;
  * "catch syscall" will set a catchpoint that breaks at every call/return from a system call
* Set hardware breakpoint in GDB: hbreak 
* Set watchpoint (data breakpoint) in GDB: __watch__ only break on write, __rwatch__ break on read, __awatch__ break on read/write
* Set temporary variable: set &lt;variable name&gt; = &lt;value&gt;
  * set command can be used to change the flags in EFLAGS. You just need to know the bit position of the flag you wanted to change 
    + For example to set the zero flag, first set a temporary variable: set $ZF = 6 (bit position 6 in EFLAGS is zero flag). Use that variable to set the zero flag bit: set $eflags |= (1 << $ZF)
    + To figure out the bit position of a flag that you are interested in, check out this image below:
    
<p align='center'> <img src="http://css.csail.mit.edu/6.858/2013/readings/i386/fig2-8.gif"> </p> 
<!-- EFLAGS Register - MIT course 6.858 --!>

#
## *<p align='center'> WinDBG Tips </p>*
* __Notations__: 
  * __?__: evaluates an expression 
  * __??__: evaluates a C++ expression
  * __!__: prefixed to tell debugger that the token is a symbol and not an expression
    * symbol can also be prefixed with module name (e.g. &lt;module&gt;!&lt;symbol&gt;) to save debugger time from searching through all modules for the matching symbol
  * __$__: prefixed to all pseudo-registers (e.g. $ip, $peb) 
  * __@__: prefixed to tell debugger that the token is a register or pseudo-register to save it time from doing symbol lookup
* WinDBG will break in the kernel, not at the entry point. Ways to find entry point: 
  * lm (loaded modules) to find the binary's image base. From image base, here are 2 ways to get entry point: 
    * !dh (dump headers) to read the image's header information to get the entry point
    * $iment to display just the entry point and not the rest of the header information 
  * $exentry: a pseudo-register that contains the entry point
* __Useful General Commands__: 
  * poi(address): displays data pointed to by address
  * d[b/w/d/q/yb/a/u/f/D/p] address L&lt;num&gt;: displays memory. The num right next to L is the range specifier that specifies the amount to display
    * dd deadbeef L4 will display 4 4-bytes values starting from address deadbeef 
  * e[b|d|D|f|p|q|w] address [values]: edits memory
    * ed deadbeef 0x10101010 0x20202020 will replace 2 4-bytes values starting from address deadbeef with 0x10101010 and then 0x20202020
  * __~__: lists all threads. ~Ns switches to the Nth thread
  * __|__: lists current process and all child processes. |Ns switches to the Nth process
  * sx(e/d/r/i): controls how the debugger handle exceptions or events
    * sxe: breaks on an event 
    * sxd: disables break for an event 
    * sxr: shows output for an event 
    * sxi: ignores an event 
---

# .instruction-sets

## *<p align='center'> x86 </p>*
* The 6 32-bit selector registers for x86 architecture: CS, DS, ES, FS, GS, SS. A selector register indicates a specific block of memory from which one can read or write. The real memory address is looked up in an internal CPU table 
  + Selector registers usually points to OS specific information. For example, FS segment register points to the beginning of current Thread Environment Block (TEB), also know as Thread Information Block (TIB), on Windows. Offset zero in TEB is the head of a linked list of pointers to exception handler functions on 32-bit system. Offset 30h is the PEB structure. Offset 2 in the PEB is the BeingDebugged field. In x64, PEB is located at offset 60h of the gs segment register
* Control register: EFLAGS. EFLAGS is a 32-bit register. It contains values of 32 boolean flags that indicate results from executing the previous instruction. EFLAGS is used by JCC instructions to decide whether to jump or not
* Calling Conventions (x86): 
  + __CDECL__: arguments pushed on stack from right to left. Caller cleaned up stack after
  + __STDCALL__: arguments pushed on stack from right to left. Callee cleaned up stack after
  + __FASTCALL__: first two arguments passed in ECX and EDX. If there are more, they are pushed onto the stack
* The call instruction contains a 32-bit signed relative displacement that is added to the address immediately following the call instruction to calculate the call destination
* __Jump Instruction__: 
  * __Short (Near) Jump__: like call instruction uses relative addressing, but with only an 8-bit signed relative displacement
  * __Long (Near) Jump__: uses larger offset value but also uses relative addressing from instruction pointer
  * __Far Jump__: uses absolute addresssing to jump to a location in a different segment. Needs to specify the segment to jump to and also the offsets from that segment 
* x86 instruction set does not provide EIP-relative data access the way it does for control-flow instructions. Thus to do EIP-relative data access, a general-purpose register must first be loaded with EIP
* The one byte NOP instruction is an alias mnemonic for the XCHG EAX, EAX instruction, although their opcodes are different
* An opcode can have multiple mnemonics associated with it and a mnemonic can have multiple opcodes associated with it
  * __Example 1__: 0x75 is both the opcode for JNZ and JNE
  * __Example 2__: 0xb142 and 0xc6c142 both corresponds to the instruction MOV CL, 66
* There is no way to tell the datatype of something stored in memory by just looking at the location of where it is stored. The datatype is implied by the operations that are used on it. For example, if an instruction loads a value into EAX, comparison is taken place between EAX and 0x10, and JA is used to jump to another location if EAX is greater, then we know that the value is an unsigned int since JA is for unsigned numbers
* EIP can only be changed through CALL, JMP, or RET
* __Floating Point Arithmetic__: Floating point operations are performed using the FPU Register Stack, or the "x87 Stack." FPU is divided into 8 registers, st0 to st7. Typical FPU operations will pop item(s) off the stack, perform on it/them, and push the result back to the stack
  + FLD instruction is for loading values onto the FPU Register Stack
  + FST instruction is for storing values from ST0 into memory 
  + FPU Register Stack can be accessed only by FPU instructions
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
* All addresses and pointers are 64-bit, but virtual addresses must be in canonical form. Canonical form means that bit 47 and bits 48-63 must match since modern processors only support 48-bit for address space rather than the full 64-bit that is available. If the address is not in canonical form, an exception will be raised 
* 16 general-purpose registers each 64-bits (RAX, RCX, RDX, RBX, RSP, RBP, RSI, RDI, R8, R9, R10, R11, R12, R13, R14, R15)
  + DWORD (32-bit) version can be accessed with a D suffix, WORD (16-bit) with a W suffix, BYTE (8-bit) with a B suffix for registers R8 to R15
  + For registers with alternate names like x86 (e.g. RAX, RCX), size access for register is same as x86. For example, 32-bit version of RAX is EAX and the 16-bit version is DX 
* 16 XMM registers, each 128-bit long. XMM registers are for SIMD instruction set, which is an extension to the x86-64 architecture. SIMD is for performing the same instruction on multiple data at once and/or for floating point operations 
  + Floating point operations were once done using stack-based instruction set that accesses the FPU Register Stack. But now, it can be done using SIMD instruction set 
* Supports instruction pointer-relative addressing on data. Unlike x86, referencing data will not use absolute address but rather an offset from RIP
* Calling conventions: Parameters are passed to registers. Additional one are stored on stack
  + Windows: first 4 parameters are placed in RCX, RDX, R8, and R9
  + Linux: first 6 parameters are placed in RDI, RSI, RDX, RCX, R8, and R9
* In 32-bit code, stack space can be allocated and unallocated in middle of the function using push or pop. However, in 64-bit code, functions cannot allocate any space in the middle of the function
* Non-leaf functions are sometimes called frame functions because they require a stack frame. Since stack space on x86-64 cannot change in the middle of a function,  all nonleaf functions are required to allocate 0x20 bytes of stack space when they call a function. This allows the function being called to save the register parameters (RCX, RDX, R8, and R9) in that space. If a function has any local stack variables, it will allocate space for them in addition to the 0x20 bytes
* __Exception Handling__:  
  * __Windows__: on x86, Structured Exception Handling (SEH) is stored on the stack, which makes it vulnerable to buffer overflow attacks. SEH on x64 is implemented differently. SEH is table-based and no longer stored on the stack. It is instead stored in the PE Header
  * __Linux__: for both x86 and x64, exception handling can be achieved through signals 
* Easier in 64-bit code to differentiate between pointers and data values. The most common size for storing integers is 32 bits and pointers are always 64 bits
* RBP is treated like another GPR. As a result, local variables are referenced through RSP
#
## *<p align='center'> ARM </p>*
* ARMv7 uses 3 profiles (Application, Real-Time, Microcontroller) and model name (Cortex). For example, ARMv7 Cortex-M is meant for microcontroller and support Thumb-2 execution only 
* Thumb-1 is used in ARMv6 and earlier. Its instructions are always 2 bytes in size
* Thumb-2 is used in ARMv7. Its instructions can be either 2 bytes or 4 bytes in size. 4 bytes Thumb instruction has a .W suffix, otherwise it generates a 2 byte Thumb instruction
* Native ARM instructions are always 4 bytes in size
* Privileges separation are defined by 8 modes. In comparison to x86, User (USR) mode is like ring3 and Supervisor (SVC) mode is like ring0
* Control register is the current program status register (CPSR), also known as application program status register (APSR), which is basically an extended EFLAGS register in x86
* There are 16 32-bit general-purpose registers (R0 - R15), but only the first 12 registers are for general purpose usage
  + R0 holds the return value from function call
  + R13 is the stack pointer (SP)
  + R14 is the link register (LR), which holds return address for function call
  + R15 is the program counter (PC)
* Only load/store instructions can access memory. All other instructions operate on registers 
  + load/store instructions: LDR/STR, LDM/STM, and PUSH/POP
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
* PUSH/POP's form: PUSH/POP {register(s)}
* PUSH/POP and STMFD/LDMFD are functionally the same, but PUSH/POP is used as prologue and epilogue in Thumb state while STMFD/LDMFD is used as prologue and epilogue in ARM state. 
* Instructions for function invocation: B, BX, BL, and BLX
  + B's syntax: B imm. imm is relative offset from R15, the program counter
  + BX's syntax: BX &lt;register&gt;. X means that it can switch between ARM and THUMB state. If the LSB of the destination is 1, it will execute in Thumb state. BX LR is commonly used to return from function 
  + BL's syntax: BL imm. It stores return address, the next instruction, in LR before transferring control to destination. To return from function, B LR is used
  + BLX's syntax: BLX imm./&lt;register&gt;. When BLX uses an offset, it always swap state (ARM to THUMB or THUMB to ARM)
* Since instructions can only be 2 or 4 bytes in size, it's not possible to directly use a 32-bit constant as an operand. As a result, barrel shifter can be used to transform the immediate into a larger value 
* For arithmetic operations, the "S" suffix will update conditional flags. Whereas, comparison instructions (CBZ, CMP, TST, CMN, and TEQ) automatically update the flags
* Instructions can be conditionally executed by adding conditional suffixes
  * This is how conditional branch instruction is implemented. For example, BEQ executes Branch instruction if EQ condition is true (ZF = 1)
* Thumb instruction cannot be conditionally executed, with the exception of B instruction, without the IT instruction. 
  + IT (If-then)'s syntax: ITxyz cc. cc is the conditional suffix for the 1st instruction after IT. xyz are for the 2nd, 3rd, and 4th instructions after IT. It can be either T or E. T means that the condition must match cc for it to be executed. E means that condition must be the opposite of cc for it to be executed
---

# .languages

## *<p align='center'> C++ Reversing </p>*
* C++ calling convention for this pointer is called thiscall: 
  + On Microsoft Visual C++ compiled binary, this is stored in ecx. Sometimes esi 
  + On g++ compiled binary, this is passed in as the first parameter of the member function as an address 
  + Class member functions are called with the usual function parameters in the stack and with ecx pointing to the class’s object 
* Child classes inherit functions and data from parent classes
* Class’s object in assembly only contains the vfptr (pointer to virtual functions table) and variables. Member functions are not part of it
  + Child class automatically has all virtual functions and data from parent class
  + Even if the programmer did not explicit write the constructor for the class. If the class contains virtual functions, a call to the constructor will be made to fill in the vfptr to point to vtable. If the class inherit from another class, within the constructor there will have a call to the constructor of the parent class  
  + vtable of a class is only referenced directly within the class constructor and destructor
  + Compiler places a pointer immediately prior to the class vtable. It points to a structure that contains information on the name of class that owns the vtable
* Memory spaces for global objects are allocated at compile-time and placed in data or bss section of binary 
* Use Name Mangling to support Method Overloading (multiple functions with same name but accept different parameters). Since in PE or ELF format, a function is only labeled with its name 
#
## *<p align='center'> Python Reversing </p>*
* PVM (Python Virtual Machine) is a stack-based virtual machine that stores operands in an upwardly-growing stack
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
<p align='center'> <img src="https://upload.wikimedia.org/wikipedia/commons/7/77/Elf-layout--en.svg" height="400"> </p>
<!-- this image is from wikipedia -->

* What's important for linking compared to what's important for execution: 

<p align='center'> <img src="https://lh4.googleusercontent.com/bdHe2FFeRH120O3iEV-Ikqs6UpTDOg-rJ7iAssHIOQEx-6XlPPeLISFYAaVB6-BL5aFL-D9y0wB9dfEz3NcTiroT6L-Q-EzPeXDT6VB1iK-BtPXub3o" height="400"> </p>
<!-- this image is found on Google -->

* ELF file header starts at offset 0 and is the roadmap that describes the rest of the file. It marks the ELF type, architecture, execution entry point, and offsets to program headers and section headers
* Program header table let the system knows how to create the process image. It contains an array of structures, each describing a segment. A segment contains one or more sections
* Section header table is not necessary for program execution. It is mainly for linking and debugging purposes. It is an array of ELF_32Shdr or ELF_64Shdr structures (Section Header)
* Relocatable objects have no program headers since they are not meant to be loaded into memory directly
* .got (Global Offset Table) section: a table of addresses located in the data section. It allows PIC code to reference data that were not available during compilation (ex: extern "var"). That data will have a section in .got, which will then be filled in later by the dynamic linker
* .plt (Procedure Linkage Table) section: a part of the text section, consisting of external function entries. Each plt entry has a correcponding entry in .got.plt which contains the actual offset to the function. Resolving of external functions is done through lazy binding. It means that it doesn't resolve the address until the function is called. 
* plt entry consists of: 
  + A jump to an address specified in GOT
  + argument to tell the resolver which function to resolve (only reach there during function's first invocation)
  + call the resolver (resides at PLT entry 0)
* .got.plt section: contains dynamically-linked function entries that can be resolved lazily
* If you compile with the -g option, the compiled binary will contain extra sections with names that start with .debug_. The most important one of the .debug section is .debug_info. It tells you the path of the source file, path of the compilation directory, version of C used, and the line numbers where variables are declared in source code. It will also maintain the parameter names for local functions
* If you compile with the -s option, the compiled binary will not contain symbol table and relocation information. This means that the .symtab will be stripped away, which contains references to variable and local function names. The dynsym section, containing references to unresolved dynamically linked functions, remains because it is needed for program execution
* The -O3 option is the second highest optimization level. The optimizations that it applied will actually result in more bytes than compiled version of the unoptimized binary
* The -funroll-loops option unroll the looping structure of any loops, making it harder for reverse engineer to analyze the compiled binary
* dlsym and dlopen can be used to dynamically resolved function names. This way those library functions won't show up on the Import Table
* __Stripped Binary__: there are 2 sections that contain symbols: .dynsym and .symtab. .dynsym contains dynamic/global symbols, those symbols are resolved at runtime. .symtab contains all the symbols
  * nm command to list all symbols in the binary from .symtab
  * Stripped binary == no .symtab symbol table
  * .dynsym symbol table cannot be stripped since it is needed for runtime, so imported library symbols remain in a stripped binary. But if a binary is compiled statically, it will have no symbol table at all if stripped
  * With non-stripped, gdb can identify local function names and knows the bounds of all functions so we can do: disas "function name"
  * With stripped binary, gdb can’t even identify main. Can identify entry point using the command: info file. Also, can’t do disas since gdb does not know the bounds of the functions so it does not know which address range should be disassembled. Solution: use examine(x) command on address pointed by pc register like: x/14i $pc
* Ways to analyze it: 
  + display section headers: readelf -S
  + display program headers and section to segment mapping: readelf -l
  + display symbol tables: readelf --syms or objdump -t
  + display a section's content: objdump -s -j <-section name-> <-binary file->
  + trace library call: ltrace -f
  + trace sys call: strace -f
  + decompile: retargetable decompiler
  + view a running program's process address space: /proc/$pid/maps
#
## *<p align='center'> PE Files </p>*

<p align='center'> <img src="http://nagareshwar.securityxploded.com/wp-content/uploads/2013/10/PE-architecture.jpg" height="400"> </p>
<!-- this image is from http://nagareshwar.securityxploded.com -->

* How a PE file is loaded into memory: 

<p align='center'> <img src="https://e16ae89e-a-62cb3a1a-s-sites.googlegroups.com/site/delphibasics/home/delphibasicsarticles/anin-depthlookintothewin32portableexecutablefileformat-part1/PEFormat1.gif?attachauth=ANoY7crq-jlujmE8zWi60Cm1Xi_rNPgeQUC1YYKQlSvboVrM-HQoeheT2P4WBCI0_ucUP_NvHGSqE8JlQMo_t8bF3lUsnZSRzHEC1uVP0Z-1v-jkIOQqVKSJpK_ryOoQDHKu3zLerXHhxlpgIXAKSSGXmsH4ysNQiSiubcM4BTBAQfJiGDhinfcUL3kWQieZD91oSDlrYJo9HEsEnULu1X9Wlcc40V77IvtcQ_eZJKq9hd4Qy42gbBBav2rxu2dBgfqxFZ-NMhK9m4oupLnWQLLWBMxf3jZoUiSsO3VeIz7yIfnX0PCj_iowkY8_lcDMgl4NQGDgehgBqvi9jn59u51cwjB9fE065A%3D%3D&attredirects=0" height="400"> </p>
<!-- this image is from DelphiBasics -->

* __Virtual Address(VA) to File Offset Translation__: file_offset = VA - image_base - section_base_RVA + section_file_offset
  1. VA - image_base = RVA. RVA (Relative Virtual Address) is virtual address relative to the image base (HMODULE). It is used to avoid hardcoded memory addresses since the image base might not always get loaded to its preferred load address. As a result, address obtained from disassembler might not match the address obtained from a debugger
  2. RVA - section_base_RVA = offset from base of the section
  3. offset_from_section_base + section_file_offset = file offset on disk  
* PE file starts with a DOS header. The 2 fields of interest in the DOS header are e_magic and e_lfanew. e_magic contains the magic number 0x5A4D (MZ). e_lfanew contains PE header's file offset
  * e_lfanew field is necessary since between DOS Header and PE Header is the DOS Stub, which for backward compatibility prints "This program cannot be run in DOS mode" if a 32-bit PE file is ran in a 16-bit DOS environment
* PE header (IMAGE_NT_HEADERS) contains 3 fields
  * __Signature__: always 0x00004550 ("PE00")
  * __IMAGE_FILE_HEADER (COFF Header)__: contains basic information on the file (e.g. target CPU, number of sections)
  * __IMAGE_OPTIONAL_HEADER32 (PE Optional Header)__: contains most of the meaningful information. Can be further broken down into 3 parts:
    * __Standard Fields__: notable fields are MagicNumber and AddressOfEntryPoint
      * __MagicNumber__: determines whether the image file uses 64-bit address space (PE32+) or 32-bit address space. PE32+ contained widened PE Optional Header
      * __AddressOfEntryPoint__: RVA to the entry point function
    * __Windows Specific Fields (Additional Fields)__: notable fields are ImageBase, SectionAlignment, and SizeOfImage
      * __ImageBase__: preferred address when loaded into memory
        * __Base Relocations__: pointed by IMAGE_DATA_DIRECTORY's entry IMAGE_DIRECTORY_ENTRY_BASERELOC. Refers to as the .reloc section. Contains every location that needs to be rebased if the executable doesn't load at the preferred load address
          * To rebase, loader calculates the difference between the actual load address and the preferred load address. Every entry in this section is then updated with the previous calculation
      * __SectionAlignment__: alignment for section when loaded into memory such that a section's VA will always be a multiple of this value 
      * __SizeOfImage__: size of image starting from image base to the last section rounded to the nearest multiple of SectionAlignment
    * __Data Directory (IMAGE_DATA_DIRECTION)__: locations of many important data structures (e.g. imports, exports, base relocations). 
      * __Program Exception Data__: pointed by IMAGE_DATA_DIRECTORY's entry IMAGE_DIRECTORY_ENTRY_EXCEPTION. It is an exception table that contains an array of IMAGE_RUNTIME_FUNCTION_ENTRY structures. Each IMAGE_RUNTIME_FUNCTION_ENTRY contains the address to an exception handler
* Section table (IMAGE_SECTION_HEADERs) is right after PE Header. Each IMAGE_SECTION_HEADER contains information on a section, such as a section's virtual address or its pointer to data on disk 
  * 2 sections can be merged into a single one if they have similar attributes
    * .idata is often merged into .rdata in release-mode executable 
* __Overlay__: data appended to end of a PE File. It is not loaded into memory
---

# .operating-system-concepts

## *<p align='center'> Windows OS </p>*
* Windows debug symbol information isn't stored inside the executable like Linux's ELF executable, where debug symbol information has its own section in the executable. Instead, it is stored in the program database (PDB) file
  + To load the PDB File along with the executable (assuming they are in the same directory): File -> Load File -> PDB File
* __Device Driver__: allows third-party developers to run code in the Windows kernel. Located in the kernel. Device drivers create/destroy device objects. User space application interacts with the driver by sending requests to a device object
* __SEH (Structured Exception Handler)__: 32-bit Windows' mechanism for handling exceptions
  * SEH chain is a list of exception handlers within a thread 
  * Each handler can choose to handle the exception or pass to the next one. If the exception made it to the last handler, it is an unhandled exception
  * FS segment register points to the Thread Environment Block (TEB). The first element of TEB is a pointer to the SEH chain
  * SEH chains is a linked list of data structures called EXCEPTION_REGISTRATION records 
  * struct _EXCEPTION_REGISTRATION {
  DWORD prev;
  DWORD handler;
  };
  * To add our own exception handler:
    + push handler
    + push fs:[0]
    + mov fs:[0], esp
* __Handles__: like pointers in that they refer to an object. It is an abstraction that hides a real memory address from the API user, allowing the system to reorganize physical memory transparently to the program
* __Windows Registry (hierarchical database of information)__: used to store OS and program configuration information. Nearly all Windows configuration information is stored in the registry, including networking, driver, startup, user account, and other information 
  + The registry is divided into five top-level sections called root keys
  + __HKEY_LOCAL_MACHINE(HKLM)__: stores settings that are global to the local machine 
  + __HKEY_CURRENT_USER(HKCU)__: stores settings specific to the current user
* DLL files look almost exactly like EXE files. For example, it also uses PE file format. The only real difference is that DLL has more exports than imports
  + Main DLL function is DllMain. It has no label and is not an export in the DLL but is specified in the PE header as the file's entry point
* Before Windows OS switches between threads, all values in the CPU are saved in a structure called the thread context. The OS then loads the thread context of the new thread into the CPU and executes the new thread
* __Pool Memory__: memory allocated by kernel-mode code
  + __Paged Pool__: memory that can be paged out
  + __Non-Paged Pool__: memory that can never be paged out
* __Memory Descriptor Lists (MDL)__: a data structure that describes the mapping between a process's virtual address to a set of physical pages. Each MDL entry describes one contiguous buffer that can be locked (can't be reused by another process) and mapped to a process's virtual address space 
* __Kernal32dll__: interface that provides APIs to interact with Windows OS
* __Ntdll__: interface to kernel. Lowest userland API
  + Native applications are applications that issue calls directly to the Natice API(Ntdll)
* __Windows API's Invocation Pipeline__: User Code -> Kernel32 with functions that end with A (e.g. CreateFileA) -> Kernel32 with functions that end with W (e.g. CreateFileW) -> Ntdll -> Kernel(ntoskrnl.exe, ...)
  + There are two versions of Kernel32 API calls if the call takes in a string: One that ends in A and one that ends in W. A for ASCII and W for wide string
  + In Kernel32 one has the option to call the API with ASCII or wide string. But if one calls it with ASCII, Windows will internally convert it to wide string and call the wide string version of the API
  + Windows API uses stdcall for its calling convention
* List of available system calls is stored in KiServiceTable (which can be found inside the KeServiceDescriptorTable structure) and every system call has an unique number that indexes into it
* System calls can be implemented using software interrupts, such as SYSENTER on x86 or SYSCALL on x86-64
  + On pre-Pentinum 2 processors, APIs call will eventually trigger int 0x2e, where index 0x2e in the IDT is the system call dispatcher
  + SYSCALL invokes system call dispatcher (KiSystemCall64) by loading RIP from IA32_LSTAR MSR (Model Specific Register) 
  + SYSENTER invokes system call dispatcher (KiFastCallEntry) by loading EIP from MSR 0x176
  + System call dispatcher will use the value in EAX to index KiServiceTable for the system call, dispatch the system call, and return to user code
#
## *<p align='center'> Interrupts </p>*
* Hardware interrupts are generated by hardware devices/peripherals (asynchronous: can happen at any time)
* Software interrupts (exceptions) are generated by the executing code and can be categorized as either faults or traps (synchronous)
  + fault is a correctable exception such as page fault. After the exception is handled, execution returns to the instruction that causes the fault
  + trap is an exception caused by executing special instruction, such as int 0x80. After the trap is handled, execution returns to the instruction after the the trap
    * int 0x80 is used to make system call. Parameters are passed through GPRs 
      + syscall number: eax 
      + 1st parameter: ebx 
      + 2nd parameter: ecx
      + 3rd parameter: edx
      + 4th parameter: esi 
      + 5th parameter: edi 
      + 6th parameter: ebp 
      + int 0x80 is an old way to make syscall on x86. A more modern implementation is the SYSENTER instruction
* When an interrupt or exception occurs, the processor uses the interrupt number as index into the Interrupt Descriptor Table (IDT), where each entry is a 8 byte KIDTENTRY structure. Each KIDTENTRY structure contains the address of a specific interrupt handler. Base of IDT is stored in the IDT register, which can be accessed through the SIDT instruction
* Interrupt is the communication between CPU and the kernel. The kernel can also notify the running process that an interrupt event has been fired by sending a signal to the process
  + For example, when a breakpoint is hit (INT3), the process will receive the SIGTRAP signal. The signal can then be handled in user code by user-defined signal handler 
* __Interrupt Requests Level (IRQL)__: an interrupt is associated with an IRQL, which indicates its priority
  + IRQL is per-processor. The local interrupt controller (LAPIC) in the processor controls task priority register (TPR) and read-only processor priority register (PPR). Code running in certain PPR level will only fire interrupts that have priority higher than PPR
---

# .anti-analysis

## *<p align='center'> Obfuscation </p>*
* Program transformation techniques that output a program that is semantically equivalent to the original program but is more difficult to analyze
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
  * __readlink (Linux)__: calling readlink on "/proc/ppid/exe" will return a string containing the location of the debugger if one is attached. You can find ppid by checking the dynamic file /proc/pid/status. And to find the pid of the process, use the ps command
    * __/proc/pid/status__: this dynamic file also contains other information on a running process, such as whether or not the process is being traced. If the field tracerPid is 0, the process is not being traced
  * Under GDB, argv[0] (name of the current invoked program) contains binary's absolute path even if you invoke the binary from a relative path. Under normal execution, argv[0] will contain the relative path. You can take advantage of this with any string-related functions, such as strcmp() and strstr(), to detect presence of GDB
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
* __Timing Checks__:  record a timestamp, perform some operations, take another timestamp, and then compare the two timestamps. If there is a lag, you can assume the presence of a debugger
  * __rdtsc Instruction (0x0F31)__: this instruction returns the count of the number of ticks since the last system reboot as a 64-bit value placed into EDX:EAX. Simply execute this instruction twice and compare the difference between the two readings
* __Detection Before main()__: to make it harder to find anti-debugging checks, anti-debugging checks can be placed in code that executes before main()
  * __TLS Callbacks__: (Windows only) Most debuggers start at the program’s entry point as defined by the PE header. TlsCallback is traditionally used to initialze thread-specific data before a thread runs, so TlsCallback is called before the entry point and therefore can execute secretly in a debugger
  * In C, function using the "constructor" attribute will execute before main()
#
## *<p align='center'> Anti-Emulation </p>*
* Using emulation allows reverse engineer to bypass many anti-debugging techniques
* __Detection through Syscall__: invoke various uncommon syscalls and check if it contains expected value. Since there are OS features not properly implemented, it means that the process is running under emulation
* __CPU Inconsistencies Detection__: try executing privileged instructions in user mode. If it succeeded, then it is under emulation
  + WRMSR is a privileged instruction (Ring 0) that is used to write values to a MSR register. Values in MSR registers are very important. For example, the SYSCALL instruction invokes the system-call handler by loading RIP from IA32_LSTAR MSR. As a result, WRMSR instruction cannot be executed in user-mode  
* __Timing Delays__: execution under emulation will be slower than running under real CPU
* __Number of Cores__: the number of cores under emulation could be smaller than the number of cores on the host machine 
#
## *<p align='center'> Anti-Dumping </p>*
* __Header Erase__: erasing the file header during program execution will prevent the program from being dumped. Even if a tool is able to dump it, the dumped image will be missing important information that the loader needs
* __Stolen Bytes__: 
#
## *<p align='center'> Bonus </p>*
* "From an anti-reversing prespective, code doesn't have to be hard to reverse engineer....all we really need in the end of the day is we need the reverse engineer give up" - Chris Domas 
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
* All forms of content modification for the purpose of hiding intent
* __Caesar Cipher__: formed by shifting the letters of alphabet #’s characters to the left or right
* __Single-Byte XOR Encoding__: modifies each byte of plaintext by performing a logical XOR operation with a static byte value
  * __Identifying XOR Loop__: looks for a small loop that contains the XOR function (where it is xor-ing a register and a constant or a register with another register)
  * __Single-byte XOR's Weakness__: if there are many null bytes then key will be easy to figure out since XOR-ing nulls with the key reveals the key. 
  * __Solutions To Single-Byte XOR Encoding's Weakness__: 
    + Null-preserving single-byte XOR encoding: if plaintext is NULL or key itself, then it will not be encoded via XOR
    + Generates the keystream used to XOR the data using a pseudorandom number generator 
      * __Blum Blum Shub PRNG__: generic form: Value<sup>i+1</sup> = (Value<sup>i</sup> * Value<sup>i</sup>) % M. M is a constant  that is the product of 2 large primes and an initial V needs to be given. Actual key being xor-ed with the data is the lowest byte of current PRNG value
* __Other Simple Encoding Scheme__:
  + ADD, SUB
  + ROL, ROR: Instructions rotate the bits within a byte right or left
  + Multibyte: XOR key is multibyte
  + Chained or loopback: Use content itself as part of the key. EX: the original key is applied at one side of the plaintext, and the encoded output character is used as the key for the next character
* __Data Encoding Example (Base64)__:

<p align='center'> <img src="http://www.tcpipguide.com/free/diagrams/mimebase64.png"> </p> 

  * Encodes binary data into character set of 64 ASCII characters
  * Most common character set is MIME’s Base64, whose table consists of A-Z, a-z, and 0-9 for the first 62 values and + / for the last 2 values
  * Base64 operates every 3 bytes (24 bits). For every 6 bits, it indexes the table with 64 characters. The encoded value is the character that is indexed with the 6 bits 
  * One padding character may be presented at the end of the encoded string (typically =) since Base64 operates every 3 bytes
  * Easy to develop a custom substitution cipher using Base64 since the only item that needs to be changed is the indexing string table of 64 characters

[Go to .table-of-contents](#table-of-contents)
