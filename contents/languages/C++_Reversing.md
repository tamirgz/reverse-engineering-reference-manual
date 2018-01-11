### [.languages](languages.md)[__C++ Reversing__]

---
#### *<p align='center'> thiscall </p>*
---
* __Thiscall__: C++'s calling convention
  + On Microsoft Visual C++ compiled binary, this pointer is stored in ecx. Sometimes esi 
  + On g++ compiled binary, this pointer is passed in as the first parameter of the member function as an address 
  + Class member functions are called with the usual function parameters in the stack and with ecx pointing to the class’s object 

---
#### *<p align='center'> How An Object Is Represented </p>*
---
* __How An Object Is Represented__: Class’s object in assembly only contains the vfptr (pointer to virtual functions table) and variables. Member functions are not part of it
  + Child class automatically has all virtual functions and data from parent class
  + Even if the programmer did not explicit write the constructor for the class. If the class contains virtual functions, a call to the constructor will be made to fill in the vfptr to point to vtable. If the class inherit from another class, within the constructor there will have a call to the constructor of the parent class  
  + vtable of a class is only referenced directly within the class constructor and destructor
  + Compiler places a pointer immediately prior to the class vtable. It points to a structure that contains information on the name of class that owns the vtable

---
#### *<p align='center'> Name Mangling </p>*
---
* __Name Mangling__ is used to support Method Overloading (multiple functions with same name but accept different parameters). Since in PE or ELF format, a function is only labeled with its name
