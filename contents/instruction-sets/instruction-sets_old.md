# .instruction-sets

## *<p align='center'> x86 </p>*
* __Registers__: temporary storage locations that are built into the CPU. Aside from the General Purpose Registers (GPRs), most other registers are dedicated to specific purposes
  * The 6 32-bit selector registers for x86 architecture: CS, DS, ES, FS, GS, SS. A selector register contains address to a specific block of memory from which one can read or write. The real memory address is looked up in an internal CPU table 
    + Selector registers usually points to OS specific information. For example, FS segment register points to the beginning of current Thread Environment Block (TEB), also know as Thread Information Block (TIB), on Windows. Offset zero in TEB is the head of a linked list of pointers to exception handler functions on 32-bit system. Offset 30h is the Process Environment Block (PEB) structure. In x64, PEB is located at offset 60h of the gs segment register
  * Control register: EFLAGS. It contains flag values that indicate results from executing the previous instruction. EFLAGS is used by JCC instructions to decide whether to jump or not
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
* __Lost Of Type Information__: there is no way to tell the datatype of something stored in memory by just looking at the location of where it is stored. The datatype is implied by the operations that are used on it. For example, if an instruction loads a value into EAX, comparison is taken place between EAX and 0x10, and JA is used to jump to another location if EAX is greater, then we know that the value is an unsigned int since JA is a jump instruction for unsigned numbers
* __Floating Point Arithmetic__: floating point operations are performed using the FPU Register Stack, or the "x87 Stack." FPU is divided into 8 registers, st0 to st7. Typical FPU operations will pop item(s) off the stack, perform on it/them, and push the result back to the stack
  + FLD instruction is for loading values onto the FPU Register Stack
  + FST instruction is for storing values from ST0 into memory 
  + FPU Register Stack can be accessed only by FPU instructions
* __Variable-Length Instruction__: an instruction's size in x86 can range from 1 to 15 bytes

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/instruction-sets/x86/x86.png" height="600" width="400"> 
<p align='center'><sub><strong>one byte x86 instructions</strong></sub></p>
</div>

* __Commonly Used But Hard To Remember x86 Instructions With Side Effects__:
  * __Side Effects?__: how executing a particular instruction will affect memory and registers
  * __IMUL reg/mem__: register is multiplied with AL, AX, or EAX and the result is stored in AX, DX:AX, or EDX:EAX
  * __IDIV reg/mem__: takes one parameter (divisor). Depending on the divisor’s size, div will use either AX, DX:AX, or EDX:EAX as the dividend, and the resulting quotient/remainder pair are stored in AL/AH, AX/DX, or EAX/EDX
  * __STOS(B/W/D)__: writes the value AL/AX/EAX to EDI. Commonly used to initialize a buffer to a constant value
  * __SCAS(B/W/D)__: compares AL/AX/EAX with data starting at the memory address EDI
  * __LODS(B/W/D)__: reads 1, 2, or 4 byte value from esi and stores it in al, ax, or eax 
  * __MOVS(B/W/D)__: moves data with 1, 2, or 4 byte granularity between two memory addresses. They implicitly use EDI/ESI as the destination/source address
  * __CLD/STD__: CLD/STD clears/sets direction flag (DF). If DF is 1, addresses are decremented. It is used by STOS(B/W/D), SCAS(B/W/D), LODS(B/W/D), and MOVS(B/W/D)  
  * __REP__: repeats an instruction up to ECX times
  * __PUSHAD/POPAD__: pushes/pops all 8 general-purpose registers 
  * __PUSHFD/POPFD__: pushes/pops EFLAGS register 
  * __MOVSX/MOVZX__: both works like a MOV except MOVSX sign-extends the value in the destination register while MOVZX zero-extends the value in the destination register   
  * __CMOVcc__: if the condition code's (cc) corresponding flag is set in EFLAGS, MOV will be performed. Otherwises, it's just like a NOP 
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
