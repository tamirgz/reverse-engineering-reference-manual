### [.tools](tools.md)[__IDA Tips__]

---
#### *<p align='center'> Addresses Shown In IDA </p>*
---
* When IDA loads a binary, it simulates a mapping of the file in memory. The addresses shown in IDA are the virtual memory addresses and not offsets of the binary file on disk
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

---
#### *<p align='center'> Import Address Table (IAT) </p>*
---
* IAT shows you all the dynamically linked libraries' functions that the binary uses. IAT is important for a reverser to understand how the binary will be interacting with the OS. To hide API calls from displaying in the IAT, a programmer can dynamically resolve them
* __How To Find Dynamically Resolved APIs__: get the binary's function trace (e.g. hybrid-analysis, ltrace). If any of the APIs in the function trace is not in the IAT, then that API is dynamically resolved
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

---
#### *<p align='center'> Saving Memory Snapshot From Your Debugger Session </p>*
---
* Debugger -> Take Memory Snapshot -> All Segments

---
#### *<p align='center'> Useful Shortcuts </p>*
---
* __u__ to undefine 
* __d__ to turn it to data 
* __c__ to turn it to code 
* __g__ to bring up the jump to address menu
* __n__ to rename
* __x__ to show cross-references

#
<p align='center'><a href="/contents/tools/tools.md">.tools</a> ~> <a href="GDB_Tips.md">GDB_Tips</a></p>
