## *<p align='center'>.tools</p>*

__The Power Of Two__
* When reversing a target, you will likely run various basic static analysis, dynamic analysis, and/or automation tools to identify point of interests for further manual analysis. Once you identified all the point of interests, the majority of your time will be spent inside a disassembler and a debugger to try to figure out if any of those point of interests leads you closer to your goal (e.g. finding exploitable bugs). As a result, at least __know how to use a disassembler and a debugger well__.

__Be Cautious...__
* Never be too reliant on any one tool. For every popular tool, depending on the tool's usage, there are ways to detect its presence (e.g. if gdb is detected, divert from normal execution), hide certain program properties from it, or make it not function properly. 

---
### *<p align='center'> section overview </p>*
---
* [IDA Tips](IDA_Tips.md)
  * [Addresses Shown In IDA](IDA_Tips.md)
  * [Import Address Table (IAT)](IDA_Tips.md)
  * [Saving Memory Snapshot From Your Debugger Session](IDA_Tips.md)
  * [Useful Shortcuts](IDA_Tips.md)
* [GDB Tips](GDB_Tips.md)
  * [Changing Default Settings](GDB_Tips.md)
  * [User Inputs](GDB_Tips.md)
  * [Automation](GDB_Tips.md)
  * [Ways To Pause Debuggee](GDB_Tips.md)
  * [Useful Commands](GDB_Tips.md)

#
<p align='center'><a href="/contents/general-knowledge/int_0x7374617274.md">int_0x7374617274</a> | <a href="/contents/tools/IDA_Tips.md">IDA_Tips</a></p>
