
---

## Working with Memory in FORTH

In this tutorial, we'll explore the process of manipulating memory in FORTH. Using a simple example, we'll create a memory space, define words to examine it, add references, and more.

### 1. Creating Memory Space
```forth
CREATE TARGET 64 ALLOT
```
Here, we define a word `TARGET` and allocate 64 bytes of memory for it. We then clear (initialize with zeros) all those bytes using the `ERASE` word.
```forth
TARGET 64 ERASE
```

### 2. Examining the Memory Space
This step introduces a word named `REVIEW` that enables you to visualize the contents of the memory space. The `REVIEW` word displays the first 32 bytes in the `TARGET` address.
```forth
: .## ( c) 0 <# # # #> TYPE SPACE ; ( Numb2String Display Format )
: REVIEW
    BASE @ >R HEX
    TARGET 32 0 DO
        DUP C@ .##
        1+
    LOOP
    DROP
    R> BASE !
;
```

### 3. Adding Reference Words
The following words serve as references, enabling easier interactions with the `TARGET` address.
```forth
: T@ ( a) TARGET + @ ;
: T! ( n a) TARGET + ! ;
: TC@ ( a) TARGET + C@ ;
: TC! ( n a) TARGET + C! ;
```

### 4. Working with Target Space
Firstly, a target pointer `TP` is created. This pointer will help in indexing the bytes within the `TARGET` address space.
```forth
VARIABLE TP ( Target Pointer) 1 TP !
: THERE ( - a) TP @ ;
```
We then define the size of a target cell with `TCELL`, set at 2 bytes.
```forth
2 CONSTANT TCELL ( 2 bytes per target cell)
```
Then, two more words `TC,` and `T,` are defined to insert values into the target space using the target pointer. The word `RENEW` serves as a convenient way to clear the `TARGET` and reset the pointer.
```forth
: TC, ( n) THERE TC! 1 TP +! ;
: T, ( n) THERE T! TCELL TP +! ;
: RENEW TARGET 64 ERASE 1 TP ! ;
```

### 5. Interacting with the Target Space
After initializing the `TARGET` with zeroes using `REVIEW`, we then add bytes to it.
```forth
REVIEW
1 TC, 2 TC, 3 TC,
REVIEW
```

### Full Code with Comments:
```forth
CREATE TARGET 64 ALLOT
TARGET 64 ERASE

\ Lets add a word to examine our target:
: .## ( c) 0 <# # # #> TYPE SPACE ; ( Numb2String Display Format )
: REVIEW
    BASE @ >R HEX
    TARGET 32 0 DO
        DUP C@ .##
        1+
    LOOP
    DROP
    R> BASE !
;

\ Lets add some references to the Target address
: T@ ( a) TARGET + @ ;
: T! ( n a) TARGET + ! ;
: TC@ ( a) TARGET + C@ ;
: TC! ( n a) TARGET + C! ;

\ Lets add some words to work in the target space
VARIABLE TP ( Target Pointer) 1 TP !
: THERE ( - a) TP @ ;

\ TP Initiated at 1, as 0 is a flag in FORTH.
2 CONSTANT TCELL ( 2 bytes per target cell)

: TC, ( n) THERE TC! 1 TP +! ;
: T, ( n) THERE T! TCELL TP +! ;
: RENEW TARGET 64 ERASE 1 TP ! ;
```

---

That concludes our tutorial on working with memory in FORTH. We hope it helped in understanding the intricacies and capabilities of FORTH when it comes to memory management.

---
