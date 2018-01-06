# .file-formats

## *<p align='center'> ELF Files </p>*

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/file-formats/ELF_Files/elf_file_format.png" height="400"> 
<p align='center'><sub><strong>ELF file format overview</strong></sub></p>
</div>

* What's important for linking compared to what's important for execution: 

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/file-formats/ELF_Files/loading_elf_file.png" height="400"> 
<p align='center'><sub><strong>linking vs execution view</strong></sub></p>
</div>

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

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/file-formats/PE_Files/pe_header.png"> 
<p align='center'><sub><strong>PE file format overview</strong></sub></p>
</div>

* How a PE file is loaded into memory: 

<div align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/file-formats/PE_Files/loading_pe_file.png"> 
<p align='center'><sub><strong>PE file in memory</strong></sub></p>
</div>

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
