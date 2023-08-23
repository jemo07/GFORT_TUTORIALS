
## A Simple Tutorial on Using `>number` in gforth

### Introduction

In gforth and ANSI Forth, `>number` is a powerful word used to convert a string representation of a number into its actual numeric value. This tutorial will guide you through the basics of using `>number` and understanding its behavior correctly.

### Basics

1. The current numeric `BASE` determines how the string is interpreted.
2. The stack requires:
   - An initial double-number accumulator (usually created from a single precision number like `0` using `s>d`).
   - The address and length of the string to be converted.

### Steps to Convert a String to a Number

1. **Set the Numeric Base (Optional)**: If you wish to convert a number in a specific base (like hexadecimal or octal), set the base using `DECIMAL`, `HEX`, or `OCTAL`.

   ```forth
   DECIMAL  \ Set the numeric base to decimal
   ```

2. **Push an Initial Double-Number Accumulator**: Use `s>d` to convert a single precision number to double precision.

   ```forth
   0 s>d  \ Converts 0 to double precision and pushes to the stack
   ```

3. **Push the String to be Converted**: Use `S" your-number-string"`.

   ```forth
   S" 1234"  \ Pushes address and length of the string "1234" onto the stack
   ```

4. **Use `>number` to Convert**: Call `>number`. It will update the accumulator with the numeric value.

   ```forth
   >number
   ```

   After the conversion, the stack will contain:
   - The updated accumulator with the numeric value.
   - The updated address and length of the string, showing how much of the string was processed.

5. **Cleaning Up**: Typically, you would drop the updated address and length, leaving the converted number on the stack.

   ```forth
   2DROP  \ Drops the updated address and length
   ```

### Complete Example

```forth
: convert-string-to-number
   0 s>d        \ Convert 0 to double precision for the accumulator
   S" 1234"     \ Push the string address and length
   >number      \ Convert the string to a number
   2DROP        \ Discard the updated address and length
   DROP         \ Drop the high word of the double accumulator, leaving the actual number
;

convert-string-to-number .  \ This will display "1234"
```

---

### Definition of `>number` in gforth and ANSI Forth

`>number` ( ud1 c-addr1 u1 – ud2 c-addr2 u2 ) core “to-number”

Attempt to convert the character string c-addr1 u1 to an unsigned number in the current number base. The double ud1 accumulates the result of the conversion to form ud2. Conversion continues, left-to-right, until the whole string is converted or a character that is not convertible in the current number base is encountered (including + or -). For each convertible character, ud1 is first multiplied by the value in BASE and then incremented by the value represented by the character. c-addr2 is the location of the first unconverted character (past the end of the string if the whole string was converted). u2 is the number of unconverted characters in the string. Overflow is not detected.

### Conclusion

Understanding the correct usage of `>number` in gforth and ANSI Forth is essential. With the above definition in mind, one can appreciate the intricate process of string-to-number conversion in Forth, allowing for a deeper understanding of its functionality and nuances.
