# <p align='center'> Reverse Engineering Reference Manual (beta) </p>

<p align='center'> 
<img src="https://github.com/yellowbyte/reverse-engineering-reference-manual/blob/master/images/heading/Introduction.PNG"> 
</p>

# .table-of-contents

* [.general-knowledge](contents/general-knowledge/general-knowledge.md)
  * [int 0x7374617274](contents/general-knowledge/int_0x7374617274.md)
    * [Threads](contents/general-knowledge/int_0x7374617274.md#-threads-)
    * [start IS NOT main](contents/general-knowledge/int_0x7374617274.md#-start-is-not-main-)
    * [Random Number Generator](contents/general-knowledge/int_0x7374617274.md#-random-number-generator-)
    * [Endianness](contents/general-knowledge/int_0x7374617274.md#-endianness-)
    * [Software Breakpoint](contents/general-knowledge/int_0x7374617274.md#-software-breakpoint-)
    * [Hardware Breakpoint](contents/general-knowledge/int_0x7374617274.md#-hardware-breakpoint-)
    * [Memory Breakpoint](contents/general-knowledge/int_0x7374617274.md#-memory-breakpoint-)
* [.tools](contents/tools/tools.md)
  * [IDA Tips](contents/tools/IDA_Tips.md)
    * [Addresses Shown In IDA](contents/tools/IDA_Tips.md#-addresses-shown-in-ida-)
    * [Import Address Table (IAT)](contents/tools/IDA_Tips.md#-import-address-table-iat-)
    * [Saving Memory Snapshot From Your Debugger Session](contents/tools/IDA_Tips.md#-saving-memory-snapshot-from-your-debugger-session-)
    * [Useful Shortcuts](contents/tools/IDA_Tips.md#-useful-shortcuts-)
  * [GDB Tips](contents/tools/GDB_Tips.md)
    * [Changing Default Settings](contents/tools/GDB_Tips.md#-changing-default-settings-)
    * [User Inputs](contents/tools/GDB_Tips.md#-user-inputs-)
    * [Automation](contents/tools/GDB_Tips.md#-automation-)
    * [Ways To Pause Debuggee](contents/tools/GDB_Tips.md#-ways-to-pause-debuggee-)
    * [Useful Commands](contents/tools/GDB_Tips.md#-useful-commands-)
* [.instruction-sets](contents/instruction-sets/instruction-sets.md)
  * [x86](#-x86-)
  * [x86-64](#-x86-64-)
  * [ARM](#-arm-)
* [.languages](contents/languages/languages.md)
  * [C++ Reversing](#-c-reversing-)
  * [Python Reversing](#-python-reversing-)
* [.file-formats](contents/file-formats/file-formats.md)
  * [ELF Files](#-elf-files-)
  * [PE Files](#-pe-files-)
* [.anti-analysis](contents/anti-analysis/anti-analysis.md)
  * [Obfuscation](#-obfuscation-)
  * [Anti-Disassembly](#-anti-disassembly-)
  * [Anti-Debugging](#-anti-debugging-)
  * [Anti-Emulation](#-anti-emulation-)
  * [Bonus](#-bonus-)
* [.encodings](contents/encodings/encodings.md)
  * [String Encoding](#-string-encoding-)
  * [Data Encoding](#-data-encoding-)
---

__NOTE(1)__: if you have any question or need further clarification on the content, feel free to create an issue and ask away!

__NOTE(2)__: beta? Yes. In the coming months I'm planning on adding more pictures and diagrams to the current content. Plans to add more sections will continue after revamping it.

__NOTE(3)__: images found in this reference manual are a mix of ones I created myself and ones I found on the internet. All images and their respective sources can be found under the images directory.
