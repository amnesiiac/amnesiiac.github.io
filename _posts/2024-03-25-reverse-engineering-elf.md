---
layout: post
title: "executable & linkable format intro (reverse engineering, elf)"
author: "melon"
date: 2024-03-25 19:02
categories: "2024"
tags:
  - code insight
---

in computing, the executable and linkable format, is a common standard file format
for executable files, object code, shared libraries, kernel modules and coredumps.

<hr>

### # elf binary header
check the header of an executable binary:
```text
$ readelf --file-header out
ELF Header:   E  L  F  cl dt ve os ab ty mh
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00          # 45=E, 4c=L, 46=F, prefixed with the 7f value
  Class:                             ELF64                          # cl=01 (32-bit) or cl=02 (64-bit)
  Data:                              2's complement, little endian  # dt=01 (LSB,little endian) / dt=02 (MSB,big endian)
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)         # CORE (4), DYN: shared object file, for libs (3)
                                                                      EXEC: executable, for binaries (2)
                                                                      REL: relocatable file, before linked into EXEC (1)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x400d90
  Start of program headers:          64 (bytes into file)
  Start of section headers:          99856 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         37
  Section header string table index: 36
```

use hexdump to check the header magic bytes, decoded as big endian order
```text
$ hexdump -C -n 16 out
00000000  7f 45 4c 46 02 01 01 00  00 00 00 00 00 00 00 00  |.ELF............|
00000010
```

application binary interface (ABI):
ABI is about a machine code communication in runtime between two binary parts like: application, library, OS\...
it describes how objects are saved in memory, how functions are called (calling convention), mangling\...

examine full header details by hexdump:
```text
$ hexdump -C -n 64 out
00000000  7f 45 4c 46 02 01 01 00  00 00 00 00 00 00 00 00  |.ELF............|
00000010  02 00 3e 00 01 00 00 00  90 0d 40 00 00 00 00 00  |..>.......@.....|
00000020  40 00 00 00 00 00 00 00  10 86 01 00 00 00 00 00  |@...............|
00000030  00 00 00 00 40 00 38 00  09 00 40 00 25 00 24 00  |....@.8...@.%.$.|
00000040
```

<hr>

### # file data in elf binary
elf file data consist of three parts: program headers (segments),
section headers (sections), data.
segments & sections are two complementary view of elf:
segment is for the linker to create execution;
section is for categorizing instructions and data.
depending on the purpose, the related header are used.

<p style="margin-bottom: 20px;"></p>

1 program headers (segments)  
elf file consists of 0 or more segments, which are desc about how to create
a proc/mem image for runtime execution.
segment are viewed by the kernel and mmap them into virtual address space.
in other words, it converts predefined instructions into a memory image.

why program headers are required?
kernel uses the program headers with the underlying data structure to form a process.
for a normal elf binary, program headers are required, otherwise it simply won’t run.

exmaine executable elf program header by readelf:
```text
$ readelf --program-headers out
Elf file type is EXEC (Executable file)
Entry point 0x400d90
There are 9 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000400040 0x0000000000400040  (01)
                 0x00000000000001f8 0x00000000000001f8  R E    8
  INTERP         0x0000000000000238 0x0000000000400238 0x0000000000400238  (02)
                 0x000000000000001c 0x000000000000001c  R      1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000  (03)
                 0x000000000000394c 0x000000000000394c  R E    200000
  LOAD           0x0000000000003dc8 0x0000000000603dc8 0x0000000000603dc8  (04)
                 0x000000000000031c 0x0000000000000480  RW     200000
  DYNAMIC        0x0000000000003de8 0x0000000000603de8 0x0000000000603de8  (05)
                 0x0000000000000210 0x0000000000000210  RW     8
  NOTE           0x0000000000000254 0x0000000000400254 0x0000000000400254  (06)
                 0x0000000000000044 0x0000000000000044  R      4
  GNU_EH_FRAME   0x0000000000002968 0x0000000000402968 0x0000000000402968  (07): sorted queue used by gcc,
                 0x00000000000002fc 0x00000000000002fc  R      4                 store exception handler.
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000  (08): a buffer or a scratch place,
                 0x0000000000000000 0x0000000000000000  RW     10                store stack infomation.
  GNU_RELRO      0x0000000000003dc8 0x0000000000603dc8 0x0000000000603dc8  (09)
                 0x0000000000000238 0x0000000000000238  R      1

 Section to Segment mapping (program header -> section):
  Segment Sections...
   00
   01     .interp
   02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r
          .rela.dyn .rela.plt .init .plt .text .fini .rodata .eh_frame_hdr .eh_frame .gcc_except_table
   03     .init_array .fini_array .jcr .dynamic .got .got.plt .data .bss
   04     .dynamic
   05     .note.ABI-tag .note.gnu.build-id
   06     .eh_frame_hdr
   07
   08     .init_array .fini_array .jcr .dynamic .got
```

GNU_STACK:
when a process function is started, a block is reserved.
when the function is finished, it will be marked as free again.
stack should not be executable to avoid security vulnerablilities.
however, by memory manipulation, one could refer to this executable stack and
run intended instructions.

if the GNU_STACK segment is not available, then usually an executable stack is used.
stack details can be examined by the scanelf and execstack tools:

```text
$ scanelf -e out
TYPE    STK/REL/PTL FILE
ET_EXEC RW- R-- RW- out

ET_EXEC: this is an executable ELF file
STK: stack segment, readable & writeable, but not executable.
REL: relocation segment, readable, but not writeable or executable.
PTL: text/program segment, readable & writeable, but not executable.
```

<p style="margin-bottom: 20px;"></p>

2 section headers (section)  
the section headers define all the sections in the file,
which is used for linking and relocation.
sections can be found in an elf after gcc transformed c into asm,
the asm is then converted to object file by the gnu assembler.

a segment can have zero or more sections, for executable files there are four main sections:
.text, .data, .rodata, and .bss, each of them is loaded with different access rights.

examine executable elf section header by readelf:
```text
$ readelf --section-headers out
There are 37 section headers, starting at offset 0x18610:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000400238  00000238
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note.ABI-tag     NOTE             0000000000400254  00000254
       0000000000000020  0000000000000000   A       0     0     4
  [ 3] .note.gnu.build-i NOTE             0000000000400274  00000274
       0000000000000024  0000000000000000   A       0     0     4
  [ 4] .gnu.hash         GNU_HASH         0000000000400298  00000298
       0000000000000038  0000000000000000   A       5     0     8
  [ 5] .dynsym           DYNSYM           00000000004002d0  000002d0
       00000000000002a0  0000000000000018   A       6     1     8
  [ 6] .dynstr           STRTAB           0000000000400570  00000570
       0000000000000302  0000000000000000   A       0     0     1
  [ 7] .gnu.version      VERSYM           0000000000400872  00000872
       0000000000000038  0000000000000002   A       5     0     2
  [ 8] .gnu.version_r    VERNEED          00000000004008b0  000008b0
       0000000000000080  0000000000000000   A       6     3     8
  [ 9] .rela.dyn         RELA             0000000000400930  00000930
       0000000000000048  0000000000000018   A       5     0     8
  [10] .rela.plt         RELA             0000000000400978  00000978
       0000000000000258  0000000000000018  AI       5    24     8
  [11] .init             PROGBITS         0000000000400bd0  00000bd0
       000000000000001a  0000000000000000  AX       0     0     4
  [12] .plt              PROGBITS         0000000000400bf0  00000bf0
       00000000000001a0  0000000000000010  AX       0     0     16
  [13] .text             PROGBITS         0000000000400d90  00000d90   (*)
       0000000000001a72  0000000000000000  AX       0     0     16
  [14] .fini             PROGBITS         0000000000402804  00002804
       0000000000000009  0000000000000000  AX       0     0     4
  [15] .rodata           PROGBITS         0000000000402810  00002810   (*)
       0000000000000158  0000000000000000   A       0     0     8
  [16] .eh_frame_hdr     PROGBITS         0000000000402968  00002968
       00000000000002fc  0000000000000000   A       0     0     4
  [17] .eh_frame         PROGBITS         0000000000402c68  00002c68
       0000000000000c8c  0000000000000000   A       0     0     8
  [18] .gcc_except_table PROGBITS         00000000004038f4  000038f4
       0000000000000058  0000000000000000   A       0     0     4
  [19] .init_array       INIT_ARRAY       0000000000603dc8  00003dc8
       0000000000000010  0000000000000008  WA       0     0     8
  [20] .fini_array       FINI_ARRAY       0000000000603dd8  00003dd8
       0000000000000008  0000000000000008  WA       0     0     8
  [21] .jcr              PROGBITS         0000000000603de0  00003de0
       0000000000000008  0000000000000000  WA       0     0     8
  [22] .dynamic          DYNAMIC          0000000000603de8  00003de8
       0000000000000210  0000000000000010  WA       6     0     8
  [23] .got              PROGBITS         0000000000603ff8  00003ff8
       0000000000000008  0000000000000008  WA       0     0     8
  [24] .got.plt          PROGBITS         0000000000604000  00004000
       00000000000000e0  0000000000000008  WA       0     0     8
  [25] .data             PROGBITS         00000000006040e0  000040e0   (*)
       0000000000000004  0000000000000000  WA       0     0     1
  [26] .bss              NOBITS           0000000000604100  000040e4   (*)
       0000000000000148  0000000000000000  WA       0     0     32
  [27] .comment          PROGBITS         0000000000000000  000040e4
       000000000000002d  0000000000000001  MS       0     0     1
  [28] .debug_aranges    PROGBITS         0000000000000000  00004111
       0000000000000580  0000000000000000           0     0     1
  [29] .debug_info       PROGBITS         0000000000000000  00004691
       0000000000007021  0000000000000000           0     0     1
  [30] .debug_abbrev     PROGBITS         0000000000000000  0000b6b2
       0000000000000938  0000000000000000           0     0     1
  [31] .debug_line       PROGBITS         0000000000000000  0000bfea
       0000000000000d1c  0000000000000000           0     0     1
  [32] .debug_str        PROGBITS         0000000000000000  0000cd06
       00000000000075a7  0000000000000001  MS       0     0     1
  [33] .debug_ranges     PROGBITS         0000000000000000  000142ad
       0000000000000570  0000000000000000           0     0     1
  [34] .symtab           SYMTAB           0000000000000000  00014820
       00000000000013f8  0000000000000018          35    59     8
  [35] .strtab           STRTAB           0000000000000000  00015c18
       000000000000288b  0000000000000000           0     0     1
  [36] .shstrtab         STRTAB           0000000000000000  000184a3
       0000000000000168  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
```

1) .text section: contains executable code, packed into a segment with r/e access rights.
it can be loaded only once, as the contents normally will not change, which can
be confirmed by objdump:

```text
$ objdump --section-headers out | grep -C 2 .text
 11 .plt          000001a0  0000000000400bf0  0000000000400bf0  00000bf0  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 12 .text         00001a72  0000000000400d90  0000000000400d90  00000d90  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 13 .fini         00000009  0000000000402804  0000000000402804  00002804  2**2
```

2) .data section: initialized data, with read/write access rights (WA).  
3) .rodata section: initialized data, with read access rights only (A).  
4) .bss section: uninitialized data, with read/write access rights (WA).

<p style="margin-bottom: 20px;"></p>

3 section groups  
some sections can be grouped, in other words there's a dependency between them.
newer linkers support this functionality. however, section groups occurs not that often:

examine executable elf section header by readelf:
```text
$ readelf --section-groups out
There are no section groups in this file.
```

<p style="margin-bottom: 20px;"></p>

4 symbol table of elf binary file (demangle the mangled names)
```text
$ readelf --symbols --demangle out

Symbol table '.dynsym' contains 28 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND [...]@GLIBC_2.2.5 (2)
     2: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
                                     ...
    23: 0000000000604220    32 OBJECT  WEAK   DEFAULT   26 t[...]@CXXABI_1.3 (4)
    24: 0000000000400d30     0 FUNC    GLOBAL DEFAULT  UND [...]@GLIBCXX_3.4 (3)
    25: 0000000000400c90     0 FUNC    GLOBAL DEFAULT  UND [...]@GLIBCXX_3.4 (3)
    26: 0000000000604100   272 OBJECT  GLOBAL DEFAULT   26 [...]@GLIBCXX_3.4 (3)
    27: 0000000000400d50     0 FUNC    GLOBAL DEFAULT  UND _[...]@CXXABI_1.3 (4)

Symbol table '.symtab' contains 213 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000400238     0 SECTION LOCAL  DEFAULT    1 .interp
     2: 0000000000400254     0 SECTION LOCAL  DEFAULT    2 .note.ABI-tag
     3: 0000000000400274     0 SECTION LOCAL  DEFAULT    3 .note.gnu.build-id
     4: 0000000000400298     0 SECTION LOCAL  DEFAULT    4 .gnu.hash
     5: 00000000004002d0     0 SECTION LOCAL  DEFAULT    5 .dynsym
     6: 0000000000400570     0 SECTION LOCAL  DEFAULT    6 .dynstr
                                     ...
   103: 000000000040102a    70 FUNC    WEAK   DEFAULT   13 Stub::Stub()
   104: 0000000000401f6c    65 FUNC    WEAK   DEFAULT   13 std::_Rb_tree<ch[...]
   105: 0000000000401070    42 FUNC    WEAK   DEFAULT   13 Stub::~Stub()
   106: 000000000040137e    26 FUNC    WEAK   DEFAULT   13 std::map<char*, [...]
   107: 0000000000402524   142 FUNC    WEAK   DEFAULT   13 std::_Rb_tree_no[...]
   108: 0000000000401bd6    37 FUNC    WEAK   DEFAULT   13 std::map<char*, [...]
   109: 0000000000400c90     0 FUNC    GLOBAL DEFAULT  UND _ZNSt8ios_base4I[...]
   164: 000000000040186c    26 FUNC    WEAK   DEFAULT   13 std::_Rb_tree<ch[...]
   165: 0000000000401316    56 FUNC    WEAK   DEFAULT   13 std::_Rb_tree<ch[...]
                                     ...
   166: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _ZSt28_Rb_tree_r[...]
   210: 0000000000400edf   204 FUNC    GLOBAL DEFAULT   13 main
   211: 0000000000400bd0     0 FUNC    GLOBAL DEFAULT   11 _init
   212: 0000000000401bb4    34 FUNC    WEAK   DEFAULT   13 std::_Rb_tree_it[...]
```

fyi, alpine linux package readelf support demangle option:
```text
$(host) docker run -it --rm -v /${path_to_elf}:/${elf} alpine sh
$(alpine) apk update && apk add binutils
$(alpine) readelf ...
```

<hr>

### # static binaries vs dynamic binaries
there are two types of elf binaries: static linked and dynamic linked.

the dynamic linked binaries need external components to run correctly,
and they are used for optimization purposes.
the external components are normal libraries containing common functions:
e.g., opening files or creating a network socket.

the static linked binaries have all libraries included, which is bigger but
more portable between system.

the file command can be used to check a file is statically or dynamically compiled:
```text
$ file out
out: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs),
for GNU/Linux 2.6.32, BuildID[sha1]=2b48c482f39ee23d601356325e47f01fd8fc8f09, not stripped
```

to determine the external libraries being used:
```text
$ ldd out
        linux-vdso.so.1 =>  (0x00007ffd7bd49000)
        libstdc++.so.6 => /lib64/libstdc++.so.6 (0x00007ffa07d1a000)
        libm.so.6 => /lib64/libm.so.6 (0x00007ffa07a18000)
        libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007ffa07802000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007ffa075e6000)
        libc.so.6 => /lib64/libc.so.6 (0x00007ffa07218000)
        /lib64/ld-linux-x86-64.so.2 (0x00007ffa08022000)
```

to examine the underlying dependancy:
```text
$ lddtree out
...
```

<hr>

### # conclusion
elf file is very flexible: provides support for many CPU types, machine arch,
and operating systems.
it is also very extensible: the file format can be constructed differently, depending on the
requirements.

there are some advanced tools to help dive deep in reverse engineering:
binary analysis toolkit (BAP), binary analysis next generation (BANG), 
cutter2 (graphical user interface for radare2), LIEF (libs for analysis of executable foramt),
manticore (dynamic binary analysis tool), PyREBox (python scripable reverse engineering toolbox),
radare2 (reverse engineering and binary analysis tool).

elf is a great start for those who are into malware research, or want to know better how processes
behave or not behave.
