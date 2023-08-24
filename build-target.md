from:  http://www.nm.net/~dtaliafe/bldgtcom.htm

# Building a Remote Target Compiler

At long last we reach the apex in this three-part expose that rips the lid off the obscure yet powerful embedded development technique known as remote target compilation.

In the first article we observed the `CREATE DOES>` mechanism for implementing defining words which can generate classes of child words with identical run-time behavior and variable data attributes. In the second article, it was seen that defining words can easily spawn data compiling child words, and when these words are fed into the Forth interpretation stream a custom assembler or compiler emerges in a few lines of code.

A remote Forth target compiler runs on an embedded development host machine and allows interactive production of executable Forth and assembler routines to a target microprocessor. As keyboard definitions or source files are interpreted by the host Forth, the resulting code and data is transparently uploaded to the target for immediate testing and further programming.

In this article I will demonstrate how a remote target compiler can be built using defining words, the Forth interpreter, and a few other nifty tricks. Since we have already covered the necessary groundwork, this piece will first explore the basic elements of remote Forth target compilation, then describe a 56002 DSP target compiler I wrote using a handful of Forth concepts that collectively do something amazing - interactively compile and execute code from one architecture to another. These ideas are merely an extension to the the last article, "Easy Target Compilation".

Because I used a standard macro assembler to write the 56k Forth, my target compiler had some interesting twists to true Forth target compilation, which uses a Forth assembler to generate the target nucleus. In addition to producing a target Forth kernel, the macro assembler provides a means for the host Forth to "clone" the target Forth. Using a listing of the target Forth code addresses generated during the nucleus assembly process, a new dictionary is created on the host that enables the interactive development of Forth and assembler routines on the target.

This technique makes remote target compiler construction accessable to the pedestrian Forth programmer. Public domain Forths exist in assembly form for most processors; coupled with any host Forth on their favored operating system, the development environment described here can be easily duplicated.
Some Basic Concepts of Remote Forth Target Compilation

A "resident" Forth contains the Forth interpreter and compiler entirely on the native processor that the Forth is running on. In the original eForth for the 56002, the target contained an interpreter, compiler, and name dictionary, in addition to the target executable code dictionary. This consumed about 9K processor words in the 56002. The goal of the remote target compiler concept is to move the Forth interpretation and execution tasks from the target microprocessor to the host computer. This leaves the minimal executable target Forth system needed to create and execute new applications interactively, and to act as an operating system for applications when the host computer is removed.

In other words, the embedded target includes a run-time interpreter to execute the application Forth source routines but does not include the interactive Forth interpreter/compiler. The interactive development occurs on the host machine; a run-time interpreter and the Forth kernel words on the target communicate with the host during development and allow for immediate testing of routines. When development is complete a binary image is produced for romming or loading on the target.

This opens up interesting possibilities for embedded development, for now the target system's behavior can be controlled by ascii text scripts interpreted on the host Forth. Both host and target code execution and compilation can occur within the same source code scripts.

Forth exhibits meta compilation, the ability to reproduce itself on the machine it is running on or to another computing system. For a Forth to regenerate itself to a different architecture, a target assembler implementing the target's instruction set is written (see last article, "Easy Target Compilation"), which is used to construct a Forth virtual machine for the target, which is itself used to build a high-level Forth model. This compilation process generally takes place into the host Forth's memory, producing an image of the target Forth that can be loaded into the target memory for execution.

In the 56002 target compiler, meta compilation is used to "clone" the target nucleus, which already exists in the form of the eForth macro assembly source code. The tricky task of cross meta compiling the host Forth to a target architecture is eliminated. The assembler source provides the addresses for all the words in the target Forth's dictionary, which are used to create a target clone dictionary on the host. ( This new target dictionary may be identical to the host's own Forth dictionary; depending on the state of target compilation, either dictionary may have the correct homograph. )

Meta compilation becomes simple, because now we merely use CREATE DOES> to construct a defining word that creates host clones for each target word, and then use [ to interpret those words. With the addition of some serial communication routines and the embedded Forth virtual machine, remote target compilation is easily achieved.

A minimal target Forth system would contain the core primitives (about 32 in eForth), and whatever additional words needed to support the application. In a well-tuned application this could amount to a few hundred bytes of code. Each assembly primitive consists of a few lines of machine code that implement some operation in the Forth virtual machine, such as pushing a data item to the stack or transmitting a byte out the serial interface.

One important primitive provides a threading mechanism, called the "inner interpreter", that executes the list of addresses that comprises a high level Forth definition. This primitive can be called by through a communication routine to remotely execute target Forth definitions.

Thus the component parts of a generic remote target compiler include :

  - A target Forth nucleus consisting of the inner interpreter and core primitives.

  - A "host" Forth for target program development and file management.

  - The host forth target compiler - a program running on the host that interactively compiles executable code into the target.

  - Host and target communication routines; how one Forth talks to another.

Figure 1 gives a celestial representation to the remote target compiler development system universe. 

## Simple Example : A Remote Forth Target Compiler for the 56002 DSP

### The target Forth nucleus - 56002 eForth

The Motorola 56002 is a 24-bit "digital signal processor "(DSP) with a Harvard architecture that can execute memory operation in parallel. The multiple address registers and three memory spaces allow it to operate on time sequences of input samples, the basis of digital filtering and signal processing.

I ported eForth, by C.H. Ting, to the 56002 by converting the MASM 8086 source to Motorola's freeware 56k macro assembler. Since eForth, and its descendant, hForth, are designed for porting, the task was pretty straightforward. Implementations for a number of microprocessors exist in the public domain.

At reset, a microprocessor fetches its starting code location from a specific hardware address, called the reset vector. It then jumps to that location and begins executing whatever instructions are there. From this raw beginning the system's hardware configuration is set up, after which the application code is started. On even the simplest architecture we can breathe life into silicon by having a Forth virtual machine awaiting the execution path of the microprocessor after hardware initialization. For 56002 eForth, this consists of the inner interpreter, called doLIST, and the core primitives that implement the stack-based virtual computer.

In the eForth assembly source code, macros are used to create a linked list of the Forth dictionary. The macros package each definition with a sequence of interpreter instructions that keep the chain of threaded execution resolved. These same macros also define the address for each word in the Forth dictionary, and links to the next and previous word.

The address of an executable Forth word (routine) is called the word's code address.

A high-level Forth word in memory is basically a list of data and/or the code addresses of other Forth words that make up the definition.

The code address, the first address in this list, contains a jump to the Forth inner interpreter, doLIST, the mechanism for executing the list of data and code addresses that make up the word being executed.

To boot eForth on an embedded micro, the first instructions after hardware initialization set up the Forth vm by initializing the Forth stack pointer, return pointer, and the inner interpreter pointer. To initiate the Forth machine, the code address of a high level Forth word is copied into the interpreter pointer register. The code then jumps to the location pointed to by the register, which for every high level, or "colon", Forth word, is a jump to doLIST.

For an embedded application the first word may be a control program of some sort. In a resident eForth, the first Forth instruction is COLD, which initializes the stack, the TIB, and the terminal I/O, then enters QUIT, the familiar interactive Forth interface, also known as the "outer interpreter".

To use the 56k nucleus in a target compiler system, the name dictionary was removed by modifying the assembly macros. In the target compiler, we don't need it since name interpretation takes place on the host. Only the code addresses are necessary. Most of the words that make up the high-level Forth model were also removed, leaving the primitives and math words. With the addition of simple communication routines the eForth nucleus becomes fully usable for remote target development.

### The hForth host

The host Forth system used for this project is hForth 86, by Wonyong Koh of South Korea. This is a public-domain ANSI compliant Forth that runs under MS-DOS. The kernel can be adapted to non-DOS x86 systems and other micros as well. hForth is a derivative of eForth 86, which was the model used for eForth 56 - the target kernel.

Any decent Forth can be used since the target and host do not have to be the same model. For the 68HC11, Pygmy would be a good candidate to try this out on, since the registered version ($15) comes with a metacompiler, a 68HC11 Forth assembler, and serial i/o routines.
The host Forth remote target compiler - HFTCOM56

When the 56k target Forth nucleus is assembled through the Motorola 56000 DOS assembler, the assembly macros write the ascii names for the Forth words along with their code addresses to a text file.
  
```text
[56ke4th.asm line 139]:    doLIST    code address:  A0Fh
  

[56ke4th.asm line 165]:    next      code address:  A16h
  

[56ke4th.asm line 187]:    ?branch   code address:  A28h
  
...etc.
```
    

Thus we have a listing of the 200+ words that make up the operating system/language known as Forth, as well as the target system code addresses for each word.
Send in the clones...

Using the listing from the target56.asm assembly, a Forth compiler can be produced by parsing each line for a name and code address. A host Forth _"clone"_ definition is created with the same name as the target word, and contains the code address in the host definition. Listing 1 shows the the essential target compiler.

The target compiler reads the listing file and creates an executable host _"clone"_ of each embedded target nucleus word. Observe the defining word `TCLONE` in listing 1. When this word is executed it creates a host Forth word with a parameter field value that is either a target word code address, or the address for the target `HERE` - the next available dictionary location in the target. The run-time behavior of the new child word is defined by `TCOMPILE?` :

*if the host is in target compile mode :*
   *the clone word compiles its code address into the host memory target image (where a new word is being compiled)*

*if the host is in remote mode :*
    *the clone word transmits its code address to the target for remote execution*

*if the host is in host mode :*
    *the clone word pushes its code address onto the host stack*

The host Forth system has also been modified so that stack items entered into the interpreter are either compiled into the host target image or pushed onto the target stack, depending on the state of the remote flag.

To clone the target Forth dictionary from the assembly listing file, each line of the file is parsed into a buffer, to which the string `TCLONE` is appended. The buffer is passed to `EVALUATE`, which performs the normal Forth input stream interpretation.

We now have one leg of the target compiler - a means through `TCLONE` to create executable clones of the target nucleus. The next step is to devise a means to create new target routines using the existing nucleus clones and data in Forth definitions. We again use the defining word `TCLONE`, and the Forth interpreter to create a target compiler for the target architecture. 
Host Forth interpretive "umbilical compiling" to the target

To create new target Forth definitions interactively, which is a foundation of the Forth paradigm, we can co-opt the host Forth interpreter and compiler to generate executable code for the target, even if it has a different architecture.

When a new target definition is created in the host Forth it is "umbilical compiled" to the target transparently. This means that the list of target Forth code addresses or assembly object code that comprises the new definition is linked into the target memory as soon as the definition has been correctly entered. The code is now a part of the target Forth operating system.

`T:` and `T;` from Listing 1 define the rest of the remote target compiler. To create a new target word, the user enters `T:` (pronounced "t colon") and the desired name for the new word, its definition, and `T;` ( "t semi" ), which terminates the compilation sequence.

`T: MYWORD TARGETWORD1 ....TARGETWORDn ...T;`
      
We could have named these routines ":" and ";" , since T: and T; reside in a seperate dictionary than the Forth dictionary.

A target definition buffer is created in host memory with a Motorola .lod file header and the jmp doLIST opcode. As the programmer types existing target word names into the host terminal interface they are executed by the host. On execution, they compile their code addresses into the definition buffer.

When the programmer ends his definition, (by typing `T;` ) another .lod directive is appended to the host target definition image. The image is then transmitted to the embedded target and becomes part of the target Forth.

A "clone" of the newly defined word is also created at this time, and if the host Forth is not in the target compiling state, the word can be executed on the embedded target by typing its name into the host Forth terminal interface.

As more definitions are added, the compiled code is appended to the image buffer. This buffer can be transmitted to the target or saved to a DOS .lod file for later use.

On reset the target retains the new definitions - this is useful in case the programmer locks up the target. With some additional programming, new definitions could be written into target flash ram (as on the 56002 EVM) - allowing the target to power-on autostart the new Forth application. 

### How the target compiler works :

Here again is the essential target compiler :

```forth
: TCLONE CREATE ,  DOES> TCOMPILE? ;

: T;  'TEXIT @ EXECUTE !TCOMPILE DEF>T ;

: T:  LOCATE! THERE? TCOMPILE TCLONE JMPDOLIST, [ ] ;
```  

When `T:` initiates a new target definition, `LOCATE!` sets the host's target buffer pointers. `THERE?` fetches the next available target code address and pushes it on the host stack. `TCOMPILE` sets the host to target compile mode. When the code address and name string (`MYWORD` in the example above) are passed to `TCLONE` a new host Forth definition is created whose run-time behavior (`TCOMPILE?`) is to transmit its code address to the target, compile it into the host target definition buffer, or push it to the host Forth stack.

At this point we begin compiling the new target word's definition - the assembly object code that executes on the target. This high-level Forth object code is composed of addresses of other target words or literals (embedded numeric values), arranged as a list to be executed by `doLIST`, the Forth inner interpreter. `JMPDOLIST,` compiles a jump to `doLIST` into the code address for each new word.

`[` in the `T:` definition puts Forth in interpretation state, that is, it parses the blank delimited words from the input stream and executes them.

When each clone word entered in a new target definition executes, it compiles its code address into the target definition buffer. That's all there is to it.

The compilation is actually finished before the user types in `T;` - notice that when the user hits CR after entering in a sequence of literals or previously defined Forth words the system is returned to compile mode by the word `]` (rbrack). `T;` thus appends a code address for EXIT, the target Forth primitive that ends a colon definition - the compliment to `jmp doLIST`. `!TCOMPILE` turns off target compilation so that `DEF>T` can transmit the new definition to the target.

Now you may be wondering how "literals", numeric data entered in definitions, get compiled into target code. I wondered myself, as the project deadline loomed. The host Forth outer interpreter would have to be changed so that it would understand when to target compile numbers. For instance, a definition containing hex numbers :

```forth
  / strobe the pins on port B
  T: STROBE-PORTB 0 PBD C! FF PBD C ! 0 PBD C! T;
 ``` 

The problem is that normally Forth would see the numbers, 0h and FFh, and push them on the host stack, when we need them compiled into the target. This level of Forth programming seemed beyond my reach, but luckily I had chosen hForth, which offered a simple solution (fortuitously described in the distribution readme file).

When Forth encounters a token during interpretation, it first searches the dictionary to find a word match, and on failure tries to convert it to a number. In hForth, the interpretation number conversion routines are vectored in a table. To add new number types or change Forth's behavior with numeric input, one merely has to write the handling routine and store its address in the interpretation vector table, called 'doWord.

Here is a definition that causes hForth to target compile literals :

```forth
    : targetAlso

    (doubleAlso)                    \ the normal numeric interpretation routine

    TSTATE IF                       \ if we are in target compile mode...
        'doLIT @ EXECUTE            \ compile target doLIT

         \ doubleAlso leaves 1 on stack if number is a single
         1 = IF T,                  \ if single, target compile it
             ELSE                   \ if double, target compile it
                WHERE? ROT ROT (d.) T_,
             THEN EXIT

    THEN
    REMOTE IF                       \ if we are in remote mode...

         1 = IF SPUSH EOT TX        \ if single, push on target stack
             ELSE
               (d.) DPUSH EOT TX    \ if double, push on target stack
             THEN EXIT

    THEN DROP ;

 ```   

In a target compiled definition, a literal must end up on the target stack when the routine is being executed by the target kernel. `doLIT` is a target word that extracts a literal from the memory location following doLIT and pushes it on the Forth stack.

To store the address of targetAlso in the interpretation table, we simply enter :
```forth
' targetAlso 'doWord 3 CELLS + !
```
### How the newly compiled word is remotely executed on the target :

At the beginning of a colon word is a jump to `doLIST`, which processes the word by threading through the addresses (other Forth words) in the definition. If the host is not in target compiling mode when a target clone is executed, the clone's code address is transmitted to the target instead of getting compiled into a new definition. The target system has a small monitor routine, affectionately called the "mini-interpreter", which picks up this code address and passes it to doLIST for execution. Since the mini-interpreter is the first address in the threading chain, it is the point that execution returns to when the Forth program (starting from the code address, a high level word) is finished.
Communication between host and target

## Target to host communication interface

The embedded target, in addition to containing the executable Forth nucleus words and the inner interpreter mechanism, has a small assembly level routine (the mini-interpreter) for communicating with the host computer. This routine recognizes two commands :

  1. push a 24 bit value (data or address) to the target Forth stack

  1. execute the target Forth word code address which is on the target Forth stack 

This is all that is required for complete host/target communication and control, since the routines for program loading, character I/O, etc. are all part of the target Forth nucleus, and are executed by placing their code address on the stack and remotely executing them through the mini-interpreter.

Listing 2 is the the mini-interpreter assembly source code.

The mini-interpreter employs a jump table to retrieve the command vector to execute, and recognizes single-byte ASCII commands in the range 10h - 1Fh. This allows for dumb terminal testing of the interface. Currently, three commands are in the table with room for 13 more. Character I/O is vectored to remove dependence on the serial port for host/target control. The commands :

    ^T push data to stack (must be followed by 04h - EOT)
    ^U execute address on stack 

To insert a new command, the address of the desired subroutine is written into the table. Pressing the corresponding terminal key will then execute the function. Usually, the kernel code would be re-assembled, although it is possible to modify the interpreter while it is RAM.
Host to target communication interface

On the host side are the Forth routines to communicate with the target.

Given the two commands required by the target, the host Forth can be modified to transmit code addresses and stack elements to the target. The host can emulate target routines by having the host "clones" transmit their code addresses to the target stack and executing them.

Here is the Forth source code for pushing data to the target and executing target routines. TX and RX are the PC serial port character transmit and fetch routines. EOT signals an end of transmission to the target mini-interpreter.

```forth
  \send an ascii numeric string to target
  \( addr count -- )
  : TYPE>T ?DUP IF 0 DO DUP C@ TX  CHAR+ LOOP THEN DROP ;

  \send a double to target (up to 24 bits)
  : D>T  (d.) SPILL TYPE>T ;

  \send an ascii string to target
  : $>T  SPILL COUNT TYPE>T ;

  \send a single to target (up to 16 bits)
  : S>T  S>D D>T ;

  \push a 16 bit single number to the target stack
  : SPUSH ^T TX  S>T EOT TX ;

  \push a 24 bit double number to the target stack
  : DPUSH ^T TX D>T EOT TX ;

  \execute word on target stack
  : TEXEC ^U TX ;

```
From these primitives the rest of the host-target interaction is established.

### A real-time linker?

I didn't have time to write a Forth assembler for the 56k, but wanted to demonstrate that the system could interactively target compile assembly object code. To do this, I came up with a host word, :ASM56 that shells from hForth to DOS, where the assembly module is written and assembled, and then reads the resulting object code file into hForth and links it as a Forth word into the target.

To create an assembly language routine that integrates directly into the target kernel, the user defines a new Forth word with the desired name (ex. :ASM56 FIRBLK1.ASM ).

On entering the new definition name, the host Forth :

  1. Fetches the current target code pointer. 

  1. Opens an assembler template file containing macros and hardware equates pertinent to the target system memory map. 

  1. Copies the template file, and the current target code pointer to a file named in the definition (ex. FIRBLK1.ASM) 

  1. Shells to DOS. 

At this point any DOS editor can be used to add code into the new file, which has the necessary pointers for the assembler to locate the code correctly. After the code has been edited, assembled, and converted to a COFF format file (.lod file) without errors, the user exits back to the host Forth (type c:>exit).

When the host Forth is returned, it "clones" the assembly object code into the host target dictionary, and transmits the .lod file to the target. The new executable assembly object code exists on the target, and can be executed by typing its name into the host Forth interpreter while it is in REMOTE mode. It can also be used with other target Forth words in a new T: definition.

This entire process is transparent to the user, with the exception of the editing/assembling loop. :ASM56 thus performs the duty of a real-time linker by allowing new assembly object code to be located into the target Forth operating system. 

Here is a screen dump, with additional comments, of the shell process...
```text

  :ASM56 FILTER1.ASM
  Now edit your new assembly source file : FILTER1.ASM
  ...and run it thru asm56000.exe...
  press a key to shell to dos

  \ at this point edit the file filter1.asm, assemble, and convert to a .lod file

  c:> code filter1  \ this batch file runs filter1.asm thru the assembler

  c:> fix filter1   \ this runs the strip and lod utilities

  c:> exit          \ exit DOS to hForth...

  Target word name : FILTER1
  .ASM file : FILTER1.ASM
  .LOD file : FILTER1.LOD

  \ FILTER1 is now part of the target vocabulary and can be executed by entering
  \ its name into hForth

  WORDS

  FILTER1 CTOP COLD :CODE RESET GETLINK SETLINK 'BOOT WARMhi hi ?CSP !CSP t.S
  tDUMP X Y P PMEM YMEM XMEM ~ t. U. U.R .R PACE EMIT KEY ?KEY D2F STR2FRAC
  FRACTABL DECPNT tDECIMAL tHEX FILL MOVE CMOVE @EXECUTE HERE

  ...etc
```
    

Parting thoughts

To execute filter code on the DSP, the full address register set must be available. The 56k nucleus contains a mechanism to push the Forth virtual machine context and enter assembly object code, then return to Forth. Thus filter routines called as high-level Forth words will run at assembly speed.

The different concepts that mode the compiler work are fertile ground for additional embedded tool building using Forth, and are worthwhile topics for the student or researcher. In a professional environment, one would want to opt for a complete development system from a Forth vendor.

A problem is that these packages are priced out of the reach of the curious, and therefor limit Forth's exposure.( But they still cost less than comparable embedded C development systems. A typical C cross compiler costs on the average 1-2k, and remote source level debugging tools range from $750 - $20k ).

In my latest job I have been using a VxWorks, a pricey embedded UNIX-style operating system running under a UNIX or Windows host. It includes cross-compiling, sourcle level debugging, and dynamic linking, among other features. The host side development tools are written in the Tcl scripting language, which has parallels in Forth.

The remote target compiler described in this article shares some of these features; cross-compilation, dynamic linking, and a scripting interface. The target nucleus and mini-interpreter could easily be extended to implement a C or asm remote source-level debugger for embedded micros, which could be used with the numerous C cross compilers.

Given Forth's power, the scope of such a project is fairly small. If an embedded C remote source level debugger written in Forth were made freely available to the masses of microcontroller programmers, Forth could gain greater exposure, and, I hesitate to be so bold, help return Forth to its rightful status as the language of choice for embedded programming.
The Essential Target Compiler source code
```forth

  ( compile a numeric string into target code space - used to compile 24-bit numbers )
  ( WHERE $numeric count -- )
  : T_,
           DUP 6 SWAP - >R ROT R> +
           ROT ROT
           0 DO DUP C@ ROT 2DUP C! 1+ DUP WHERE!  \ copy chars into buffer
           SWAP DROP SWAP 1+ LOOP DROP            \ and inc WHERE for ea char
           DUP BL SWAP C! 1+ WHERE!               \ insert blank after number
           1 THERE +! ;

  ( compile a 16-bit number into target space )
  ( target code addr -- )
  : T, WHERE? SWAP S>D (d.) T_, ;

  \( target code addr  -- ) execute target address and get response
  : T@EXEC BASE @ >R
          HEX
          @ SPUSH              \ fetch target word code pointer and send to target
          TEXEC IN??           \ get ascii response, if any
          R> BASE ! ;          \ restore base


  \ if we are in target compile mode, the word code addr is compiled into the defbuff
  \ else
  \ if we are in REMOTE mode, the word code addr is sent to the target
  \ if we are in HOST mode, the word code addr is pushed on the host stack

  : TCOMPILE?   TSTATE IF
                  @                 \ fetch target word code pointer
                  T,                \ compile into host target space
                ELSE
                REMOTE IF T@EXEC    \ send code address to target and execute
                       ELSE @       \ else push on host stack
                       THEN
                THEN ;

  \ compile the opcode for jmp doLIST into target space 
  : JMPDOLIST,  WHERE? 'JMP 2@ (d.)  T_, ;

  \ the 56k target compiler

  : TCLONE CREATE ,  DOES> TCOMPILE? ;

  : T;  'TEXIT @ EXECUTE !TCOMPILE DEF>T ;

  : T:  LOCATE! THERE? TCOMPILE TCLONE JMPDOLIST, [ ] ;


  ```

Mini-interpreter Assembly code


```assembly
;; the mini-interpreter

; init forth vitual machine
;   synchronize with host
;   vm_init : clear sp
;             init forth virtual machine
;
;   cmdloop : move  #cmdtable, r1
;             inchar?
;                no -> jmp cmdloop
;                yes
;                   hi nibble == 1?               ; valid commands 10h-1Fh
;                      no -> jmp cmdloop
;                      yes
;                          mask low nibble -> n1
;                          p:(r1+n1),r0
;                          jmp r0                   ; execute command
;                jmp  cmdloop




; set up forth virtual machine

vm_init         jsr     tack            ; ack forth word executed
                move    #0,sp           ; clear the stack
                move    #SPP,r7         ; init vm
                move    #RPP,r6

; To exit a primitive to NEXT or a colon to EXIT need to have loaded r5, the Forth
; instruction pointer, with address of vm_init - the forth virtual machine
; initialization.

                move    #vm_init,r5

cmdloop         move    #cmdtable,r1
                clr     a

                jsr     getchar         ; character in a0

gotchar         move    a,y0            ; save input character

                move    #>$01,x0        ; test if high nibble == 1
                rep     #4
                lsr     a
                cmp     x0,a
                jne     cmdloop         ; no - not valid character

                move    y0,a            ; yes - move low nibble into n0
                move    #>$000f,x1      ; mask out high nibble for table index
                and     x1,a
                move    a,n1
                nop

                move    p:(r1+n1),r0    ; set up mini-interpreter command jump
jump            jmp     r0              ; execute mini-interpreter command

                jmp     cmdloop



;; mini-interpreter command routines

; valid commands : (only two needed for forth kernel interface...)

;       tpush -        push data to target stack
;       execute -      execute address of forth word on target stack


tpush           jsr     indata          ; get up to 6 ascii chars and convert to number
                move    b,x:-(r7)       ; push the number on forth stack
                jmp     cmdloop         ; return to mini-interpreter

execute         move    x:(r7)+,r4      ; pop code address off forth stack
                nop
                jmp     (r4)            ; execute forth word
                jmp     cmdloop         ; return to mini-interpreter

eforth          jmp     eFORTH         ; go start resident eForth



;; mini-interpreter command table

; Command table interpreter jumps on control characters. ^P is a problem
; since it locks up hForth (print command). So we are starting with ^T thru
; ^Z as usable control chars in DOS, which puts the entries at location 4 thru
; 10 ; ascii 14h thru 1Ah. Empty positions in table vector to cmdloop.


cmdtable        dc      cmdloop                 ; 0     10h - ^P
                dc      cmdloop                 ; 1     11h - ^Q

                dc      cmdloop                 ; 2     12h - ^R
                dc      cmdloop                 ; 3     13h - ^S
                dc      tpush                   ; 4     14h - ^T
                dc      execute                 ; 5     15h - ^U

                dc      eforth                  ; 6     16h - ^V
                dc      cmdloop                 ; 7     ...     
                dc      cmdloop                 ; 8
                dc      cmdloop                 ; 9
                dc      cmdloop                 ; a
                dc      cmdloop                 ; b
                dc      cmdloop                 ; c
                dc      cmdloop                 ; d
                dc      cmdloop                 ; e
                dc      cmdloop                 ; f
```
