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
