---
title: gcc-inline-assembly-how-to
tags:[technics, programming-language, CN, translated]
categories: CS
---

翻译节选自[How to Use Inline Assembly Language in C Code](https://gcc.gnu.org/onlinedocs/gcc/Using-Assembly-Language-with-C.html)  

<!--more-->

## Basic Assembly

A basic `asm` statement has the following syntax:

```
asm asm-qualifiers ( AssemblerInstructions )
```

The `asm` keyword is a GNU extension. When writing code that can be compiled with -ansi and the various -std options, use `__asm__` instead of `asm` (see [Alternate Keywords](https://gcc.gnu.org/onlinedocs/gcc/Alternate-Keywords.html#Alternate-Keywords)).



- `volatile`

  The optional `volatile` qualifier has no effect. All basic `asm` blocks are implicitly volatile.

- `inline`

  If you use the `inline` qualifier, then for inlining purposes the size of the `asm` statement is taken as the smallest size possible (see [Size of an asm](https://gcc.gnu.org/onlinedocs/gcc/Size-of-an-asm.html#Size-of-an-asm)).



- AssemblerInstructions

  This is a literal string that specifies the assembler code. The string can contain any instructions recognized by the assembler, including directives. GCC does not parse the assembler instructions themselves and does not know what they mean or even whether they are valid assembler input.You may place multiple assembler instructions together in a single `asm` string, separated by the characters normally used in assembly code for the system. A combination that works in most places is a newline to break the line, plus a tab character (written as ‘\n\t’). Some assemblers allow semicolons as a line separator. However, note that some assembler dialects use semicolons to start a comment.



Using extended `asm` (see [Extended Asm](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#Extended-Asm)) typically produces smaller, safer, and more efficient code, and in most cases it is a better solution than basic `asm`. However, there are two situations where only basic `asm` can be used:

- Extended `asm` statements have to be inside a C function, so to write inline assembly language at file scope (“top-level”), outside of C functions, you must use basic `asm`. You can use this technique to emit assembler directives, define assembly language macros that can be invoked elsewhere in the file, or write entire functions in assembly language. Basic `asm` statements outside of functions may not use any qualifiers.
- Functions declared with the `naked` attribute also require basic `asm` (see [Function Attributes](https://gcc.gnu.org/onlinedocs/gcc/Function-Attributes.html#Function-Attributes)).

Safely accessing C data and calling functions from basic `asm` is more complex than it may appear. To access C data, it is better to use extended `asm`.

Do not expect a sequence of `asm` statements to remain perfectly consecutive after compilation. If certain instructions need to remain consecutive in the output, put them in a single multi-instruction `asm` statement. Note that GCC’s optimizers can move `asm` statements relative to other code, including across jumps.

`asm` statements may not perform jumps into other `asm` statements. GCC does not know about these jumps, and therefore cannot take account of them when deciding how to optimize. Jumps from `asm` to C labels are only supported in extended `asm`.

Under certain circumstances, GCC may duplicate (or remove duplicates of) your assembly code when optimizing. This can lead to unexpected duplicate symbol errors during compilation if your assembly code defines symbols or labels.

**Warning:** The C standards do not specify semantics for `asm`, making it a potential source of incompatibilities between compilers. These incompatibilities may not produce compiler warnings/errors.

GCC does not parse basic `asm`’s AssemblerInstructions, which means there is no way to communicate to the compiler what is happening inside them. GCC has no visibility of symbols in the `asm` and may discard them as unreferenced. It also does not know about side effects of the assembler code, such as modifications to memory or registers. Unlike some compilers, GCC assumes that no changes to general purpose registers occur. This assumption may change in a future release.

To avoid complications from future changes to the semantics and the compatibility issues between compilers, consider replacing basic `asm` with extended `asm`. See [How to convert from basic asm to extended asm](https://gcc.gnu.org/wiki/ConvertBasicAsmToExtended) for information about how to perform this conversion.

The compiler copies the assembler instructions in a basic `asm` verbatim to the assembly language output file, without processing dialects or any of the ‘%’ operators that are available with extended `asm`. This results in minor differences between basic `asm` strings and extended `asm` templates. For example, to refer to registers you might use ‘%eax’ in basic `asm` and ‘%%eax’ in extended `asm`.

On targets such as x86 that support multiple assembler dialects, all basic `asm` blocks use the assembler dialect specified by the -masm command-line option (see [x86 Options](https://gcc.gnu.org/onlinedocs/gcc/x86-Options.html#x86-Options)). Basic `asm` provides no mechanism to provide different assembler strings for different dialects.

For basic `asm` with non-empty assembler string GCC assumes the assembler block does not change any general purpose registers, but it may read or write any globally accessible variable.

Here is an example of basic `asm` for i386:

```
/* Note that this code will not compile with -masm=intel */
#define DebugBreak() asm("int $3")
```

## Extended Assembly

With extended `asm` you can read and write C variables from assembler and perform jumps from assembler code to C labels. Extended `asm` syntax uses colons (‘:’) to delimit the operand parameters after the assembler template:

```
asm asm-qualifiers ( AssemblerTemplate 
                 : OutputOperands 
                 [ : InputOperands
                 [ : Clobbers ] ])

asm asm-qualifiers ( AssemblerTemplate 
                      : 
                      : InputOperands
                      : Clobbers
                      : GotoLabels)
```

where in the last form, asm-qualifiers contains `goto` (and in the first form, not).

The `asm` keyword is a GNU extension. When writing code that can be compiled with -ansi and the various -std options, use `__asm__` instead of `asm` (see [Alternate Keywords](https://gcc.gnu.org/onlinedocs/gcc/Alternate-Keywords.html#Alternate-Keywords)).



- `volatile`

  The typical use of extended `asm` statements is to manipulate input values to produce output values. However, your `asm` statements may also produce side effects. If so, you may need to use the `volatile` qualifier to disable certain optimizations. See [Volatile](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#Volatile).

- `inline`

  If you use the `inline` qualifier, then for inlining purposes the size of the `asm` statement is taken as the smallest size possible (see [Size of an asm](https://gcc.gnu.org/onlinedocs/gcc/Size-of-an-asm.html#Size-of-an-asm)).

- `goto`

  This qualifier informs the compiler that the `asm` statement may perform a jump to one of the labels listed in the GotoLabels. See [GotoLabels](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#GotoLabels).



- AssemblerTemplate

  This is a literal string that is the template for the assembler code. It is a combination of fixed text and tokens that refer to the input, output, and goto parameters. See [AssemblerTemplate](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#AssemblerTemplate).

- OutputOperands

  A comma-separated list of the C variables modified by the instructions in the AssemblerTemplate. An empty list is permitted. See [OutputOperands](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#OutputOperands).

- InputOperands

  A comma-separated list of C expressions read by the instructions in the AssemblerTemplate. An empty list is permitted. See [InputOperands](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#InputOperands).

- Clobbers

  A comma-separated list of registers or other values changed by the AssemblerTemplate, beyond those listed as outputs. An empty list is permitted. See [Clobbers and Scratch Registers](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#Clobbers-and-Scratch-Registers).

- GotoLabels

  When you are using the `goto` form of `asm`, this section contains the list of all C labels to which the code in the AssemblerTemplate may jump. See [GotoLabels](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#GotoLabels).`asm` statements may not perform jumps into other `asm` statements, only to the listed GotoLabels. GCC’s optimizers do not know about other jumps; therefore they cannot take account of them when deciding how to optimize.

The total number of input + output + goto operands is limited to 30.



The `asm` statement allows you to include assembly instructions directly within C code. This may help you to maximize performance in time-sensitive code or to access assembly instructions that are not readily available to C programs.

Note that extended `asm` statements must be inside a function. Only basic `asm` may be outside functions (see [Basic Asm](https://gcc.gnu.org/onlinedocs/gcc/Basic-Asm.html#Basic-Asm)). Functions declared with the `naked` attribute also require basic `asm` (see [Function Attributes](https://gcc.gnu.org/onlinedocs/gcc/Function-Attributes.html#Function-Attributes)).

While the uses of `asm` are many and varied, it may help to think of an `asm` statement as a series of low-level instructions that convert input parameters to output parameters. So a simple (if not particularly useful) example for i386 using `asm` might look like this:

```
int src = 1;
int dst;   

asm ("mov %1, %0\n\t"
    "add $1, %0"
    : "=r" (dst) 
    : "r" (src));

printf("%d\n", dst);
```

This code copies `src` to `dst` and add 1 to `dst`.



#### 6.47.2.1 Volatile



This i386 code demonstrates a case that does not use (or require) the `volatile` qualifier. If it is performing assertion checking, this code uses `asm` to perform the validation. Otherwise, `dwRes` is unreferenced by any code. As a result, the optimizers can discard the `asm` statement, which in turn removes the need for the entire `DoCheck` routine. By omitting the `volatile` qualifier when it isn’t needed you allow the optimizers to produce the most efficient code possible.

```
void DoCheck(uint32_t dwSomeValue)
{
   uint32_t dwRes;

   // Assumes dwSomeValue is not zero.
   asm ("bsfl %1,%0"
     : "=r" (dwRes)
     : "r" (dwSomeValue)
     : "cc");

   assert(dwRes > 3);
}
```

The next example shows a case where the optimizers can recognize that the input (`dwSomeValue`) never changes during the execution of the function and can therefore move the `asm` outside the loop to produce more efficient code. Again, using the `volatile` qualifier disables this type of optimization.

```
void do_print(uint32_t dwSomeValue)
{
   uint32_t dwRes;

   for (uint32_t x=0; x < 5; x++)
   {
      // Assumes dwSomeValue is not zero.
      asm ("bsfl %1,%0"
        : "=r" (dwRes)
        : "r" (dwSomeValue)
        : "cc");

      printf("%u: %u %u\n", x, dwSomeValue, dwRes);
   }
}
```

The following example demonstrates a case where you need to use the `volatile` qualifier. It uses the x86 `rdtsc` instruction, which reads the computer’s time-stamp counter. Without the `volatile` qualifier, the optimizers might assume that the `asm` block will always return the same value and therefore optimize away the second call.

```
uint64_t msr;

asm volatile ( "rdtsc\n\t"    // Returns the time in EDX:EAX.
        "shl $32, %%rdx\n\t"  // Shift the upper bits left.
        "or %%rdx, %0"        // 'Or' in the lower bits.
        : "=a" (msr)
        : 
        : "rdx");

printf("msr: %llx\n", msr);

// Do other work...

// Reprint the timestamp
asm volatile ( "rdtsc\n\t"    // Returns the time in EDX:EAX.
        "shl $32, %%rdx\n\t"  // Shift the upper bits left.
        "or %%rdx, %0"        // 'Or' in the lower bits.
        : "=a" (msr)
        : 
        : "rdx");

printf("msr: %llx\n", msr);
```

GCC’s optimizers do not treat this code like the non-volatile code in the earlier examples. They do not move it out of loops or omit it on the assumption that the result from a previous call is still valid.

Note that the compiler can move even `volatile asm` instructions relative to other code, including across jump instructions. For example, on many targets there is a system register that controls the rounding mode of floating-point operations. Setting it with a `volatile asm` statement, as in the following PowerPC example, does not work reliably.

```
asm volatile("mtfsf 255, %0" : : "f" (fpenv));
sum = x + y;
```

The compiler may move the addition back before the `volatile asm` statement. To make it work as expected, add an artificial dependency to the `asm` by referencing a variable in the subsequent code, for example:

```
asm volatile ("mtfsf 255,%1" : "=X" (sum) : "f" (fpenv));
sum = x + y;
```

Under certain circumstances, GCC may duplicate (or remove duplicates of) your assembly code when optimizing. This can lead to unexpected duplicate symbol errors during compilation if your `asm` code defines symbols or labels. Using ‘%=’ (see [AssemblerTemplate](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#AssemblerTemplate)) may help resolve this problem.



#### 6.47.2.2 Assembler Template



You may place multiple assembler instructions together in a single `asm` string, separated by the characters normally used in assembly code for the system. A combination that works in most places is a newline to break the line, plus a tab character to move to the instruction field (written as ‘\n\t’). Some assemblers allow semicolons as a line separator. However, note that some assembler dialects use semicolons to start a comment.

Do not expect a sequence of `asm` statements to remain perfectly consecutive after compilation, even when you are using the `volatile` qualifier. If certain instructions need to remain consecutive in the output, put them in a single multi-instruction `asm` statement.

Accessing data from C programs without using input/output operands (such as by using global symbols directly from the assembler template) may not work as expected. Similarly, calling functions directly from an assembler template requires a detailed understanding of the target assembler and ABI.

Since GCC does not parse the assembler template, it has no visibility of any symbols it references. This may result in GCC discarding those symbols as unreferenced unless they are also listed as input, output, or goto operands.



In addition to the tokens described by the input, output, and goto operands, these tokens have special meanings in the assembler template:

- ‘%%’

  Outputs a single ‘%’ into the assembler code.

- ‘%=’

  Outputs a number that is unique to each instance of the `asm` statement in the entire compilation. This option is useful when creating local labels and referring to them multiple times in a single template that generates multiple assembler instructions.

- ‘%{’

- ‘%|’

- ‘%}’

  Outputs ‘{’, ‘|’, and ‘}’ characters (respectively) into the assembler code. When unescaped, these characters have special meaning to indicate multiple assembler dialects, as described below.



On targets such as x86, GCC supports multiple assembler dialects. The -masm option controls which dialect GCC uses as its default for inline assembler. The target-specific documentation for the -masm option contains the list of supported dialects, as well as the default dialect if the option is not specified. This information may be important to understand, since assembler code that works correctly when compiled using one dialect will likely fail if compiled using another. See [x86 Options](https://gcc.gnu.org/onlinedocs/gcc/x86-Options.html#x86-Options).

If your code needs to support multiple assembler dialects (for example, if you are writing public headers that need to support a variety of compilation options), use constructs of this form:

```
{ dialect0 | dialect1 | dialect2... }
```

This construct outputs `dialect0` when using dialect #0 to compile the code, `dialect1` for dialect #1, etc. If there are fewer alternatives within the braces than the number of dialects the compiler supports, the construct outputs nothing.

For example, if an x86 compiler supports two dialects (‘att’, ‘intel’), an assembler template such as this:

```
"bt{l %[Offset],%[Base] | %[Base],%[Offset]}; jc %l2"
```

is equivalent to one of

```
"btl %[Offset],%[Base] ; jc %l2"   /* att dialect */
"bt %[Base],%[Offset]; jc %l2"     /* intel dialect */
```

Using that same compiler, this code:

```
"xchg{l}\t{%%}ebx, %1"
```

corresponds to either

```
"xchgl\t%%ebx, %1"                 /* att dialect */
"xchg\tebx, %1"                    /* intel dialect */
```

There is no support for nesting dialect alternatives.



#### 6.47.2.3 Output Operands



In this i386 example, `old` (referred to in the template string as `%0`) and `*Base` (as `%1`) are outputs and `Offset` (`%2`) is an input:

```
bool old;

__asm__ ("btsl %2,%1\n\t" // Turn on zero-based bit #Offset in Base.
         "sbb %0,%0"      // Use the CF to calculate old.
   : "=r" (old), "+rm" (*Base)
   : "Ir" (Offset)
   : "cc");

return old;
```

Operands are separated by commas. Each operand has this format:

```
[ [asmSymbolicName] ] constraint (cvariablename)
```

- asmSymbolicName

  Specifies a symbolic name for the operand. Reference the name in the assembler template by enclosing it in square brackets (i.e. ‘%[Value]’). The scope of the name is the `asm` statement that contains the definition. Any valid C variable name is acceptable, including names already defined in the surrounding code. No two operands within the same `asm` statement can use the same symbolic name.When not using an asmSymbolicName, use the (zero-based) position of the operand in the list of operands in the assembler template. For example if there are three output operands, use ‘%0’ in the template to refer to the first, ‘%1’ for the second, and ‘%2’ for the third.

- constraint

  A string constant specifying constraints on the placement of the operand; See [Constraints](https://gcc.gnu.org/onlinedocs/gcc/Constraints.html#Constraints), for details.Output constraints must begin with either ‘=’ (a variable overwriting an existing value) or ‘+’ (when reading and writing). When using ‘=’, do not assume the location contains the existing value on entry to the `asm`, except when the operand is tied to an input; see [Input Operands](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#InputOperands).After the prefix, there must be one or more additional constraints (see [Constraints](https://gcc.gnu.org/onlinedocs/gcc/Constraints.html#Constraints)) that describe where the value resides. Common constraints include ‘r’ for register and ‘m’ for memory. When you list more than one possible location (for example, `"=rm"`), the compiler chooses the most efficient one based on the current context. If you list as many alternates as the `asm` statement allows, you permit the optimizers to produce the best possible code. If you must use a specific register, but your Machine Constraints do not provide sufficient control to select the specific register you want, local register variables may provide a solution (see [Local Register Variables](https://gcc.gnu.org/onlinedocs/gcc/Local-Register-Variables.html#Local-Register-Variables)).

- cvariablename

  Specifies a C lvalue expression to hold the output, typically a variable name. The enclosing parentheses are a required part of the syntax.

When the compiler selects the registers to use to represent the output operands, it does not use any of the clobbered registers (see [Clobbers and Scratch Registers](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#Clobbers-and-Scratch-Registers)).

Output operand expressions must be lvalues. The compiler cannot check whether the operands have data types that are reasonable for the instruction being executed. For output expressions that are not directly addressable (for example a bit-field), the constraint must allow a register. In that case, GCC uses the register as the output of the `asm`, and then stores that register into the output.

Operands using the ‘+’ constraint modifier count as two operands (that is, both as input and output) towards the total maximum of 30 operands per `asm` statement.

Use the ‘&’ constraint modifier (see [Modifiers](https://gcc.gnu.org/onlinedocs/gcc/Modifiers.html#Modifiers)) on all output operands that must not overlap an input. Otherwise, GCC may allocate the output operand in the same register as an unrelated input operand, on the assumption that the assembler code consumes its inputs before producing outputs. This assumption may be false if the assembler code actually consists of more than one instruction.

The same problem can occur if one output parameter (a) allows a register constraint and another output parameter (b) allows a memory constraint. The code generated by GCC to access the memory address in b can contain registers which *might* be shared by a, and GCC considers those registers to be inputs to the asm. As above, GCC assumes that such input registers are consumed before any outputs are written. This assumption may result in incorrect behavior if the `asm` statement writes to a before using b. Combining the ‘&’ modifier with the register constraint on a ensures that modifying a does not affect the address referenced by b. Otherwise, the location of b is undefined if a is modified before using b.

`asm` supports operand modifiers on operands (for example ‘%k2’ instead of simply ‘%2’). Typically these qualifiers are hardware dependent. The list of supported modifiers for x86 is found at [x86 Operand modifiers](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#x86Operandmodifiers).

If the C code that follows the `asm` makes no use of any of the output operands, use `volatile` for the `asm` statement to prevent the optimizers from discarding the `asm` statement as unneeded (see [Volatile](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#Volatile)).

This code makes no use of the optional asmSymbolicName. Therefore it references the first output operand as `%0` (were there a second, it would be `%1`, etc). The number of the first input operand is one greater than that of the last output operand. In this i386 example, that makes `Mask` referenced as `%1`:

```
uint32_t Mask = 1234;
uint32_t Index;

  asm ("bsfl %1, %0"
     : "=r" (Index)
     : "r" (Mask)
     : "cc");
```

That code overwrites the variable `Index` (‘=’), placing the value in a register (‘r’). Using the generic ‘r’ constraint instead of a constraint for a specific register allows the compiler to pick the register to use, which can result in more efficient code. This may not be possible if an assembler instruction requires a specific register.

The following i386 example uses the asmSymbolicName syntax. It produces the same result as the code above, but some may consider it more readable or more maintainable since reordering index numbers is not necessary when adding or removing operands. The names `aIndex` and `aMask` are only used in this example to emphasize which names get used where. It is acceptable to reuse the names `Index` and `Mask`.

```
uint32_t Mask = 1234;
uint32_t Index;

  asm ("bsfl %[aMask], %[aIndex]"
     : [aIndex] "=r" (Index)
     : [aMask] "r" (Mask)
     : "cc");
```

Here are some more examples of output operands.

```
uint32_t c = 1;
uint32_t d;
uint32_t *e = &c;

asm ("mov %[e], %[d]"
   : [d] "=rm" (d)
   : [e] "rm" (*e));
```

Here, `d` may either be in a register or in memory. Since the compiler might already have the current value of the `uint32_t` location pointed to by `e` in a register, you can enable it to choose the best location for `d` by specifying both constraints.



#### 6.47.2.4 Flag Output Operands



On some targets, a special form of output operand exists by which conditions in the flags register may be outputs of the asm. The set of conditions supported are target specific, but the general rule is that the output variable must be a scalar integer, and the value is boolean. When supported, the target defines the preprocessor symbol `__GCC_ASM_FLAG_OUTPUTS__`.

Because of the special nature of the flag output operands, the constraint may not include alternatives.

Most often, the target has only one flags register, and thus is an implied operand of many instructions. In this case, the operand should not be referenced within the assembler template via `%0` etc, as there’s no corresponding text in the assembly language.

- x86 family

  The flag output constraints for the x86 family are of the form ‘=@cccond’ where cond is one of the standard conditions defined in the ISA manual for `jcc` or `setcc`.`a`“above” or unsigned greater than`ae`“above or equal” or unsigned greater than or equal`b`“below” or unsigned less than`be`“below or equal” or unsigned less than or equal`c`carry flag set`e``z`“equal” or zero flag set`g`signed greater than`ge`signed greater than or equal`l`signed less than`le`signed less than or equal`o`overflow flag set`p`parity flag set`s`sign flag set`na``nae``nb``nbe``nc``ne``ng``nge``nl``nle``no``np``ns``nz`“not” flag, or inverted versions of those above



#### 6.47.2.5 Input Operands



Operands are separated by commas. Each operand has this format:

```
[ [asmSymbolicName] ] constraint (cexpression)
```

- asmSymbolicName

  Specifies a symbolic name for the operand. Reference the name in the assembler template by enclosing it in square brackets (i.e. ‘%[Value]’). The scope of the name is the `asm` statement that contains the definition. Any valid C variable name is acceptable, including names already defined in the surrounding code. No two operands within the same `asm` statement can use the same symbolic name.When not using an asmSymbolicName, use the (zero-based) position of the operand in the list of operands in the assembler template. For example if there are two output operands and three inputs, use ‘%2’ in the template to refer to the first input operand, ‘%3’ for the second, and ‘%4’ for the third.

- constraint

  A string constant specifying constraints on the placement of the operand; See [Constraints](https://gcc.gnu.org/onlinedocs/gcc/Constraints.html#Constraints), for details.Input constraint strings may not begin with either ‘=’ or ‘+’. When you list more than one possible location (for example, ‘"irm"’), the compiler chooses the most efficient one based on the current context. If you must use a specific register, but your Machine Constraints do not provide sufficient control to select the specific register you want, local register variables may provide a solution (see [Local Register Variables](https://gcc.gnu.org/onlinedocs/gcc/Local-Register-Variables.html#Local-Register-Variables)).Input constraints can also be digits (for example, `"0"`). This indicates that the specified input must be in the same place as the output constraint at the (zero-based) index in the output constraint list. When using asmSymbolicName syntax for the output operands, you may use these names (enclosed in brackets ‘[]’) instead of digits.

- cexpression

  This is the C variable or expression being passed to the `asm` statement as input. The enclosing parentheses are a required part of the syntax.

When the compiler selects the registers to use to represent the input operands, it does not use any of the clobbered registers (see [Clobbers and Scratch Registers](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#Clobbers-and-Scratch-Registers)).

If there are no output operands but there are input operands, place two consecutive colons where the output operands would go:

```
__asm__ ("some instructions"
   : /* No outputs. */
   : "r" (Offset / 8));
```

**Warning:** Do *not* modify the contents of input-only operands (except for inputs tied to outputs). The compiler assumes that on exit from the `asm` statement these operands contain the same values as they had before executing the statement. It is *not* possible to use clobbers to inform the compiler that the values in these inputs are changing. One common work-around is to tie the changing input variable to an output variable that never gets used. Note, however, that if the code that follows the `asm` statement makes no use of any of the output operands, the GCC optimizers may discard the `asm` statement as unneeded (see [Volatile](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#Volatile)).

`asm` supports operand modifiers on operands (for example ‘%k2’ instead of simply ‘%2’). Typically these qualifiers are hardware dependent. The list of supported modifiers for x86 is found at [x86 Operand modifiers](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#x86Operandmodifiers).

In this example using the fictitious `combine` instruction, the constraint `"0"` for input operand 1 says that it must occupy the same location as output operand 0. Only input operands may use numbers in constraints, and they must each refer to an output operand. Only a number (or the symbolic assembler name) in the constraint can guarantee that one operand is in the same place as another. The mere fact that `foo` is the value of both operands is not enough to guarantee that they are in the same place in the generated assembler code.

```
asm ("combine %2, %0" 
   : "=r" (foo) 
   : "0" (foo), "g" (bar));
```

Here is an example using symbolic names.

```
asm ("cmoveq %1, %2, %[result]" 
   : [result] "=r"(result) 
   : "r" (test), "r" (new), "[result]" (old));
```



#### 6.47.2.6 Clobbers and Scratch Registers



Clobber descriptions may not in any way overlap with an input or output operand. For example, you may not have an operand describing a register class with one member when listing that register in the clobber list. Variables declared to live in specific registers (see [Explicit Register Variables](https://gcc.gnu.org/onlinedocs/gcc/Explicit-Register-Variables.html#Explicit-Register-Variables)) and used as `asm` input or output operands must have no part mentioned in the clobber description. In particular, there is no way to specify that input operands get modified without also specifying them as output operands.

When the compiler selects which registers to use to represent input and output operands, it does not use any of the clobbered registers. As a result, clobbered registers are available for any use in the assembler code.

Another restriction is that the clobber list should not contain the stack pointer register. This is because the compiler requires the value of the stack pointer to be the same after an `asm` statement as it was on entry to the statement. However, previous versions of GCC did not enforce this rule and allowed the stack pointer to appear in the list, with unclear semantics. This behavior is deprecated and listing the stack pointer may become an error in future versions of GCC.

Here is a realistic example for the VAX showing the use of clobbered registers:

```
asm volatile ("movc3 %0, %1, %2"
                   : /* No outputs. */
                   : "g" (from), "g" (to), "g" (count)
                   : "r0", "r1", "r2", "r3", "r4", "r5", "memory");
```

Also, there are two special clobber arguments:

- `"cc"`

  The `"cc"` clobber indicates that the assembler code modifies the flags register. On some machines, GCC represents the condition codes as a specific hardware register; `"cc"` serves to name this register. On other machines, condition code handling is different, and specifying `"cc"` has no effect. But it is valid no matter what the target.

- `"memory"`

  The `"memory"` clobber tells the compiler that the assembly code performs memory reads or writes to items other than those listed in the input and output operands (for example, accessing the memory pointed to by one of the input parameters). To ensure memory contains correct values, GCC may need to flush specific register values to memory before executing the `asm`. Further, the compiler does not assume that any values read from memory before an `asm` remain unchanged after that `asm`; it reloads them as needed. Using the `"memory"` clobber effectively forms a read/write memory barrier for the compiler.Note that this clobber does not prevent the *processor* from doing speculative reads past the `asm` statement. To prevent that, you need processor-specific fence instructions.

Flushing registers to memory has performance implications and may be an issue for time-sensitive code. You can provide better information to GCC to avoid this, as shown in the following examples. At a minimum, aliasing rules allow GCC to know what memory *doesn’t* need to be flushed.

Here is a fictitious sum of squares instruction, that takes two pointers to floating point values in memory and produces a floating point register output. Notice that `x`, and `y` both appear twice in the `asm` parameters, once to specify memory accessed, and once to specify a base register used by the `asm`. You won’t normally be wasting a register by doing this as GCC can use the same register for both purposes. However, it would be foolish to use both `%1` and `%3` for `x` in this `asm` and expect them to be the same. In fact, `%3` may well not be a register. It might be a symbolic memory reference to the object pointed to by `x`.

```
asm ("sumsq %0, %1, %2"
     : "+f" (result)
     : "r" (x), "r" (y), "m" (*x), "m" (*y));
```

Here is a fictitious `*z++ = *x++ * *y++` instruction. Notice that the `x`, `y` and `z` pointer registers must be specified as input/output because the `asm` modifies them.

```
asm ("vecmul %0, %1, %2"
     : "+r" (z), "+r" (x), "+r" (y), "=m" (*z)
     : "m" (*x), "m" (*y));
```

An x86 example where the string memory argument is of unknown length.

```
asm("repne scasb"
    : "=c" (count), "+D" (p)
    : "m" (*(const char (*)[]) p), "0" (-1), "a" (0));
```

If you know the above will only be reading a ten byte array then you could instead use a memory input like: `"m" (*(const char (*)[10]) p)`.

Here is an example of a PowerPC vector scale implemented in assembly, complete with vector and condition code clobbers, and some initialized offset registers that are unchanged by the `asm`.

```
void
dscal (size_t n, double *x, double alpha)
{
  asm ("/* lots of asm here */"
       : "+m" (*(double (*)[n]) x), "+&r" (n), "+b" (x)
       : "d" (alpha), "b" (32), "b" (48), "b" (64),
         "b" (80), "b" (96), "b" (112)
       : "cr0",
         "vs32","vs33","vs34","vs35","vs36","vs37","vs38","vs39",
         "vs40","vs41","vs42","vs43","vs44","vs45","vs46","vs47");
}
```

Rather than allocating fixed registers via clobbers to provide scratch registers for an `asm` statement, an alternative is to define a variable and make it an early-clobber output as with `a2` and `a3` in the example below. This gives the compiler register allocator more freedom. You can also define a variable and make it an output tied to an input as with `a0` and `a1`, tied respectively to `ap` and `lda`. Of course, with tied outputs your `asm` can’t use the input value after modifying the output register since they are one and the same register. What’s more, if you omit the early-clobber on the output, it is possible that GCC might allocate the same register to another of the inputs if GCC could prove they had the same value on entry to the `asm`. This is why `a1` has an early-clobber. Its tied input, `lda` might conceivably be known to have the value 16 and without an early-clobber share the same register as `%11`. On the other hand, `ap` can’t be the same as any of the other inputs, so an early-clobber on `a0` is not needed. It is also not desirable in this case. An early-clobber on `a0` would cause GCC to allocate a separate register for the `"m" (*(const double (*)[]) ap)` input. Note that tying an input to an output is the way to set up an initialized temporary register modified by an `asm` statement. An input not tied to an output is assumed by GCC to be unchanged, for example `"b" (16)` below sets up `%11` to 16, and GCC might use that register in following code if the value 16 happened to be needed. You can even use a normal `asm` output for a scratch if all inputs that might share the same register are consumed before the scratch is used. The VSX registers clobbered by the `asm` statement could have used this technique except for GCC’s limit on the number of `asm` parameters.

```
static void
dgemv_kernel_4x4 (long n, const double *ap, long lda,
                  const double *x, double *y, double alpha)
{
  double *a0;
  double *a1;
  double *a2;
  double *a3;

  __asm__
    (
     /* lots of asm here */
     "#n=%1 ap=%8=%12 lda=%13 x=%7=%10 y=%0=%2 alpha=%9 o16=%11\n"
     "#a0=%3 a1=%4 a2=%5 a3=%6"
     :
       "+m" (*(double (*)[n]) y),
       "+&r" (n),	// 1
       "+b" (y),	// 2
       "=b" (a0),	// 3
       "=&b" (a1),	// 4
       "=&b" (a2),	// 5
       "=&b" (a3)	// 6
     :
       "m" (*(const double (*)[n]) x),
       "m" (*(const double (*)[]) ap),
       "d" (alpha),	// 9
       "r" (x),		// 10
       "b" (16),	// 11
       "3" (ap),	// 12
       "4" (lda)	// 13
     :
       "cr0",
       "vs32","vs33","vs34","vs35","vs36","vs37",
       "vs40","vs41","vs42","vs43","vs44","vs45","vs46","vs47"
     );
}
```



#### 6.47.2.7 Goto Labels



An `asm goto` statement cannot have outputs. This is due to an internal restriction of the compiler: control transfer instructions cannot have outputs. If the assembler code does modify anything, use the `"memory"` clobber to force the optimizers to flush all register values to memory and reload them if necessary after the `asm` statement.

Also note that an `asm goto` statement is always implicitly considered volatile.

To reference a label in the assembler template, prefix it with ‘%l’ (lowercase ‘L’) followed by its (zero-based) position in GotoLabels plus the number of input operands. For example, if the `asm` has three inputs and references two labels, refer to the first label as ‘%l3’ and the second as ‘%l4’).

Alternately, you can reference labels using the actual C label name enclosed in brackets. For example, to reference a label named `carry`, you can use ‘%l[carry]’. The label must still be listed in the GotoLabels section when using this approach.

Here is an example of `asm goto` for i386:

```
asm goto (
    "btl %1, %0\n\t"
    "jc %l2"
    : /* No outputs. */
    : "r" (p1), "r" (p2) 
    : "cc" 
    : carry);

return 0;

carry:
return 1;
```

The following example shows an `asm goto` that uses a memory clobber.

```
int frob(int x)
{
  int y;
  asm goto ("frob %%r5, %1; jc %l[error]; mov (%2), %%r5"
            : /* No outputs. */
            : "r"(x), "r"(&y)
            : "r5", "memory" 
            : error);
  return y;
error:
  return -1;
}
```



#### 6.47.2.8 x86 Operand Modifiers

References to input, output, and goto operands in the assembler template of extended `asm` statements can use modifiers to affect the way the operands are formatted in the code output to the assembler. For example, the following code uses the ‘h’ and ‘b’ modifiers for x86:

```
uint16_t  num;
asm volatile ("xchg %h0, %b0" : "+a" (num) );
```

These modifiers generate this assembler code:

```
xchg %ah, %al
```

The rest of this discussion uses the following code for illustrative purposes.

```
int main()
{
   int iInt = 1;

top:

   asm volatile goto ("some assembler instructions here"
   : /* No outputs. */
   : "q" (iInt), "X" (sizeof(unsigned char) + 1), "i" (42)
   : /* No clobbers. */
   : top);
}
```

With no modifiers, this is what the output from the operands would be for the ‘att’ and ‘intel’ dialects of assembler:

| Operand | ‘att’  | ‘intel’           |
| ------- | ------ | ----------------- |
| `%0`    | `%eax` | `eax`             |
| `%1`    | `$2`   | `2`               |
| `%3`    | `$.L3` | `OFFSET FLAT:.L3` |

The table below shows the list of supported modifiers and their effects.

| Modifier | Description                                                  | Operand | ‘att’     | ‘intel’  |
| -------- | ------------------------------------------------------------ | ------- | --------- | -------- |
| `a`      | Print an absolute memory reference.                          | `%A0`   | `*%rax`   | `rax`    |
| `b`      | Print the QImode name of the register.                       | `%b0`   | `%al`     | `al`     |
| `c`      | Require a constant operand and print the constant expression with no punctuation. | `%c1`   | `2`       | `2`      |
| `E`      | Print the address in Double Integer (DImode) mode (8 bytes) when the target is 64-bit. Otherwise mode is unspecified (VOIDmode). | `%E1`   | `%(rax)`  | `[rax]`  |
| `h`      | Print the QImode name for a “high” register.                 | `%h0`   | `%ah`     | `ah`     |
| `H`      | Add 8 bytes to an offsettable memory reference. Useful when accessing the high 8 bytes of SSE values. For a memref in (%rax), it generates | `%H0`   | `8(%rax)` | `8[rax]` |
| `k`      | Print the SImode name of the register.                       | `%k0`   | `%eax`    | `eax`    |
| `l`      | Print the label name with no punctuation.                    | `%l3`   | `.L3`     | `.L3`    |
| `p`      | Print raw symbol name (without syntax-specific prefixes).    | `%p2`   | `42`      | `42`     |
| `P`      | If used for a function, print the PLT suffix and generate PIC code. For example, emit `foo@PLT` instead of ’foo’ for the function foo(). If used for a constant, drop all syntax-specific prefixes and issue the bare constant. See `p` above. |         |           |          |
| `q`      | Print the DImode name of the register.                       | `%q0`   | `%rax`    | `rax`    |
| `w`      | Print the HImode name of the register.                       | `%w0`   | `%ax`     | `ax`     |
| `z`      | Print the opcode suffix for the size of the current integer operand (one of `b`/`w`/`l`/`q`). | `%z0`   | `l`       |          |

`V` is a special modifier which prints the name of the full integer register without `%`.



#### 6.47.2.9 x86 Floating-Point `asm` Operands

On x86 targets, there are several rules on the usage of stack-like registers in the operands of an `asm`. These rules apply only to the operands that are stack-like registers:

1. Given a set of input registers that die in an

    

   ```
   asm
   ```

   , it is necessary to know which are implicitly popped by the

    

   ```
   asm
   ```

   , and which must be explicitly popped by GCC.

   An input register that is implicitly popped by the `asm` must be explicitly clobbered, unless it is constrained to match an output operand.

2. For any input register that is implicitly popped by an

    

   ```
   asm
   ```

   , it is necessary to know how to adjust the stack to compensate for the pop. If any non-popped input is closer to the top of the reg-stack than the implicitly popped register, it would not be possible to know what the stack looked like—it’s not clear how the rest of the stack “slides up”.

   All implicitly popped input registers must be closer to the top of the reg-stack than any input that is not implicitly popped.

   It is possible that if an input dies in an `asm`, the compiler might use the input register for an output reload. Consider this example:

   ```
   asm ("foo" : "=t" (a) : "f" (b));
   ```

   This code says that input `b` is not popped by the `asm`, and that the `asm` pushes a result onto the reg-stack, i.e., the stack is one deeper after the `asm` than it was before. But, it is possible that reload may think that it can use the same register for both the input and the output.

   To prevent this from happening, if any input operand uses the ‘f’ constraint, all output register constraints must use the ‘&’ early-clobber modifier.

   The example above is correctly written as:

   ```
   asm ("foo" : "=&t" (a) : "f" (b));
   ```

3. Some operands need to be in particular places on the stack. All output operands fall in this category—GCC has no other way to know which registers the outputs appear in unless you indicate this in the constraints.

   Output operands must specifically indicate which register an output appears in after an `asm`. ‘=f’ is not allowed: the operand constraints must select a class with a single register.

4. Output operands may not be “inserted” between existing stack registers. Since no 387 opcode uses a read/write operand, all output operands are dead before the

    

   ```
   asm
   ```

   , and are pushed by the

    

   ```
   asm
   ```

   . It makes no sense to push anywhere but the top of the reg-stack.

   Output operands must start at the top of the reg-stack: output operands may not “skip” a register.

5. Some `asm` statements may need extra stack space for internal calculations. This can be guaranteed by clobbering stack registers unrelated to the inputs and outputs.

This `asm` takes one input, which is internally popped, and produces two outputs.

```
asm ("fsincos" : "=t" (cos), "=u" (sin) : "0" (inp));
```

This `asm` takes two inputs, which are popped by the `fyl2xp1` opcode, and replaces them with one output. The `st(1)` clobber is necessary for the compiler to know that `fyl2xp1` pops both inputs.

```
asm ("fyl2xp1" : "=t" (result) : "0" (x), "u" (y) : "st(1)");
```

## Constraints

Here are specific details on what constraint letters you can use with `asm` operands. Constraints can say whether an operand may be in a register, and which kinds of register; whether the operand can be a memory reference, and which kinds of address; whether the operand may be an immediate constant, and which possible values it may have. Constraints can also require two operands to match. Side-effects aren’t allowed in operands of inline `asm`, unless ‘<’ or ‘>’ constraints are used, because there is no guarantee that the side effects will happen exactly once in an instruction that can update the addressing register.

### Simple Constraints

The simplest kind of constraint is a string full of letters, each of which describes one kind of operand that is permitted. Here are the letters that are allowed:

- whitespace

  Whitespace characters are ignored and can be inserted at any position except the first. This enables each alternative for different operands to be visually aligned in the machine description even if they have different number of constraints and modifiers.

- ‘m’

  A memory operand is allowed, with any kind of address that the machine supports in general. Note that the letter used for the general memory constraint can be re-defined by a back end using the `TARGET_MEM_CONSTRAINT` macro.

- ‘o’

  A memory operand is allowed, but only if the address is *offsettable*. This means that adding a small integer (actually, the width in bytes of the operand, as determined by its machine mode) may be added to the address and the result is also a valid memory address.For example, an address which is constant is offsettable; so is an address that is the sum of a register and a constant (as long as a slightly larger constant is also within the range of address-offsets supported by the machine); but an autoincrement or autodecrement address is not offsettable. More complicated indirect/indexed addresses may or may not be offsettable depending on the other addressing modes that the machine supports.Note that in an output operand which can be matched by another operand, the constraint letter ‘o’ is valid only when accompanied by both ‘<’ (if the target machine has predecrement addressing) and ‘>’ (if the target machine has preincrement addressing).

- ‘V’

  A memory operand that is not offsettable. In other words, anything that would fit the ‘m’ constraint but not the ‘o’ constraint.

- ‘<’

  A memory operand with autodecrement addressing (either predecrement or postdecrement) is allowed. In inline `asm` this constraint is only allowed if the operand is used exactly once in an instruction that can handle the side effects. Not using an operand with ‘<’ in constraint string in the inline `asm` pattern at all or using it in multiple instructions isn’t valid, because the side effects wouldn’t be performed or would be performed more than once. Furthermore, on some targets the operand with ‘<’ in constraint string must be accompanied by special instruction suffixes like `%U0` instruction suffix on PowerPC or `%P0` on IA-64.

- ‘>’

  A memory operand with autoincrement addressing (either preincrement or postincrement) is allowed. In inline `asm` the same restrictions as for ‘<’ apply.

- ‘r’

  A register operand is allowed provided that it is in a general register.

- ‘i’

  An immediate integer operand (one with constant value) is allowed. This includes symbolic constants whose values will be known only at assembly time or later.

- ‘n’

  An immediate integer operand with a known numeric value is allowed. Many systems cannot support assembly-time constants for operands less than a word wide. Constraints for these operands should use ‘n’ rather than ‘i’.

- ‘I’, ‘J’, ‘K’, … ‘P’

  Other letters in the range ‘I’ through ‘P’ may be defined in a machine-dependent fashion to permit immediate integer operands with explicit integer values in specified ranges. For example, on the 68000, ‘I’ is defined to stand for the range of values 1 to 8. This is the range permitted as a shift count in the shift instructions.

- ‘E’

  An immediate floating operand (expression code `const_double`) is allowed, but only if the target floating point format is the same as that of the host machine (on which the compiler is running).

- ‘F’

  An immediate floating operand (expression code `const_double` or `const_vector`) is allowed.

- ‘G’, ‘H’

  ‘G’ and ‘H’ may be defined in a machine-dependent fashion to permit immediate floating operands in particular ranges of values.

- ‘s’

  An immediate integer operand whose value is not an explicit integer is allowed.This might appear strange; if an insn allows a constant operand with a value not known at compile time, it certainly must allow any known value. So why use ‘s’ instead of ‘i’? Sometimes it allows better code to be generated.For example, on the 68000 in a fullword instruction it is possible to use an immediate operand; but if the immediate value is between -128 and 127, better code results from loading the value into a register and using the register. This is because the load into the register can be done with a ‘moveq’ instruction. We arrange for this to happen by defining the letter ‘K’ to mean “any integer outside the range -128 to 127”, and then specifying ‘Ks’ in the operand constraints.

- ‘g’

  Any register, memory or immediate integer operand is allowed, except for registers that are not general registers.

- ‘X’

  Any operand whatsoever is allowed.

- ‘0’, ‘1’, ‘2’, … ‘9’

  An operand that matches the specified operand number is allowed. If a digit is used together with letters within the same alternative, the digit should come last.This number is allowed to be more than a single digit. If multiple digits are encountered consecutively, they are interpreted as a single decimal integer. There is scant chance for ambiguity, since to-date it has never been desirable that ‘10’ be interpreted as matching either operand 1 *or* operand 0. Should this be desired, one can use multiple alternatives instead.This is called a *matching constraint* and what it really means is that the assembler has only a single operand that fills two roles which `asm` distinguishes. For example, an add instruction uses two input operands and an output operand, but on most CISC machines an add instruction really has only two operands, one of them an input-output operand:`addl #35,r12 `Matching constraints are used in these circumstances. More precisely, the two operands that match must include one input-only operand and one output-only operand. Moreover, the digit must be a smaller number than the number of the operand that uses it in the constraint.

- ‘p’

  An operand that is a valid memory address is allowed. This is for “load address” and “push address” instructions.‘p’ in the constraint must be accompanied by `address_operand` as the predicate in the `match_operand`. This predicate interprets the mode specified in the `match_operand` as the mode of the memory reference for which the address would be valid.

- other-letters

  Other letters can be defined in machine-dependent fashion to stand for particular classes of registers or other arbitrary operand types. ‘d’, ‘a’ and ‘f’ are defined on the 68000/68020 to stand for data, address and floating point registers.

### Multi-Alternative

Sometimes a single instruction has multiple alternative sets of possible operands. For example, on the 68000, a logical-or instruction can combine register or an immediate value into memory, or it can combine any kind of operand into a register; but it cannot combine one memory location into another.

These constraints are represented as multiple alternatives. An alternative can be described by a series of letters for each operand. The overall constraint for an operand is made from the letters for this operand from the first alternative, a comma, the letters for this operand from the second alternative, a comma, and so on until the last alternative. All operands for a single instruction must have the same number of alternatives.

So the first alternative for the 68000’s logical-or could be written as `"+m" (output) : "ir" (input)`. The second could be `"+r" (output): "irm" (input)`. However, the fact that two memory locations cannot be used in a single instruction prevents simply using `"+rm" (output) : "irm" (input)`. Using multi-alternatives, this might be written as `"+m,r" (output) : "ir,irm" (input)`. This describes all the available alternatives to the compiler, allowing it to choose the most efficient one for the current conditions.

There is no way within the template to determine which alternative was chosen. However you may be able to wrap your `asm` statements with builtins such as `__builtin_constant_p` to achieve the desired results.

### Modifiers

Here are constraint modifier characters.

- ‘=’

  Means that this operand is written to by this instruction: the previous value is discarded and replaced by new data.

- ‘+’

  Means that this operand is both read and written by the instruction.When the compiler fixes up the operands to satisfy the constraints, it needs to know which operands are read by the instruction and which are written by it. ‘=’ identifies an operand which is only written; ‘+’ identifies an operand that is both read and written; all other operands are assumed to only be read.If you specify ‘=’ or ‘+’ in a constraint, you put it in the first character of the constraint string.

- ‘&’

  Means (in a particular alternative) that this operand is an *earlyclobber* operand, which is written before the instruction is finished using the input operands. Therefore, this operand may not lie in a register that is read by the instruction or as part of any memory address.‘&’ applies only to the alternative in which it is written. In constraints with multiple alternatives, sometimes one alternative requires ‘&’ while others do not. See, for example, the ‘movdf’ insn of the 68000.A operand which is read by the instruction can be tied to an earlyclobber operand if its only use as an input occurs before the early result is written. Adding alternatives of this form often allows GCC to produce better code when only some of the read operands can be affected by the earlyclobber. See, for example, the ‘mulsi3’ insn of the ARM.Furthermore, if the *earlyclobber* operand is also a read/write operand, then that operand is written only after it’s used.‘&’ does not obviate the need to write ‘=’ or ‘+’. As *earlyclobber* operands are always written, a read-only *earlyclobber* operand is ill-formed and will be rejected by the compiler.

- ‘%’

  Declares the instruction to be commutative for this operand and the following operand. This means that the compiler may interchange the two operands if that is the cheapest way to make all operands fit the constraints. ‘%’ applies to all alternatives and must appear as the first character in the constraint. Only read-only operands can use ‘%’.GCC can only handle one commutative pair in an asm; if you use more, the compiler may fail. Note that you need not use the modifier if the two alternatives are strictly identical; this would only waste time in the reload pass.

### [Machine Constraints](https://gcc.gnu.org/onlinedocs/gcc/Machine-Constraints.html#Machine-Constraints) 

## Assembly Labels

You can specify the name to be used in the assembler code for a C function or variable by writing the `asm` (or `__asm__`) keyword after the declarator. It is up to you to make sure that the assembler names you choose do not conflict with any other assembler symbols, or reference registers.



This sample shows how to specify the assembler name for data:

```
int foo asm ("myfoo") = 2;
```

This specifies that the name to be used for the variable `foo` in the assembler code should be ‘myfoo’ rather than the usual ‘_foo’.

On systems where an underscore is normally prepended to the name of a C variable, this feature allows you to define names for the linker that do not start with an underscore.

GCC does not support using this feature with a non-static local variable since such variables do not have assembler names. If you are trying to put the variable in a particular register, see [Explicit Register Variables](https://gcc.gnu.org/onlinedocs/gcc/Explicit-Register-Variables.html#Explicit-Register-Variables).



To specify the assembler name for functions, write a declaration for the function before its definition and put `asm` there, like this:

```
int func (int x, int y) asm ("MYFUNC");
     
int func (int x, int y)
{
   /* … */
```

This specifies that the name to be used for the function `func` in the assembler code should be `MYFUNC`.

## Explicit Register Variables

### Global Register Variables

You can define a global register variable and associate it with a specified register like this:

```
register int *foo asm ("r12");
```

Here `r12` is the name of the register that should be used. Note that this is the same syntax used for defining local register variables, but for a global variable the declaration appears outside a function. The `register` keyword is required, and cannot be combined with `static`. The register name must be a valid register name for the target platform.

Do not use type qualifiers such as `const` and `volatile`, as the outcome may be contrary to expectations. In particular, using the `volatile` qualifier does not fully prevent the compiler from optimizing accesses to the register.

Registers are a scarce resource on most systems and allowing the compiler to manage their usage usually results in the best code. However, under special circumstances it can make sense to reserve some globally. For example this may be useful in programs such as programming language interpreters that have a couple of global variables that are accessed very often.

After defining a global register variable, for the current compilation unit:

- If the register is a call-saved register, call ABI is affected: the register will not be restored in function epilogue sequences after the variable has been assigned. Therefore, functions cannot safely return to callers that assume standard ABI.
- Conversely, if the register is a call-clobbered register, making calls to functions that use standard ABI may lose contents of the variable. Such calls may be created by the compiler even if none are evident in the original program, for example when libgcc functions are used to make up for unavailable instructions.
- Accesses to the variable may be optimized as usual and the register remains available for allocation and use in any computations, provided that observable values of the variable are not affected.
- If the variable is referenced in inline assembly, the type of access must be provided to the compiler via constraints (see [Constraints](https://gcc.gnu.org/onlinedocs/gcc/Constraints.html#Constraints)). Accesses from basic asms are not supported.

Note that these points *only* apply to code that is compiled with the definition. The behavior of code that is merely linked in (for example code from libraries) is not affected.

If you want to recompile source files that do not actually use your global register variable so they do not use the specified register for any other purpose, you need not actually add the global register declaration to their source code. It suffices to specify the compiler option -ffixed-reg (see [Code Gen Options](https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html#Code-Gen-Options)) to reserve the register.



Global register variables cannot have initial values, because an executable file has no means to supply initial contents for a register.

When selecting a register, choose one that is normally saved and restored by function calls on your machine. This ensures that code which is unaware of this reservation (such as library routines) will restore it before returning.

On machines with register windows, be sure to choose a global register that is not affected magically by the function call mechanism.



When calling routines that are not aware of the reservation, be cautious if those routines call back into code which uses them. As an example, if you call the system library version of `qsort`, it may clobber your registers during execution, but (if you have selected appropriate registers) it will restore them before returning. However it will *not* restore them before calling `qsort`’s comparison function. As a result, global values will not reliably be available to the comparison function unless the `qsort` function itself is rebuilt.

Similarly, it is not safe to access the global register variables from signal handlers or from more than one thread of control. Unless you recompile them specially for the task at hand, the system library routines may temporarily use the register for other things. Furthermore, since the register is not reserved exclusively for the variable, accessing it from handlers of asynchronous signals may observe unrelated temporary values residing in the register.



### Local Register Variables

You can define a local register variable and associate it with a specified register like this:

```
register int *foo asm ("r12");
```

Here `r12` is the name of the register that should be used. Note that this is the same syntax used for defining global register variables, but for a local variable the declaration appears within a function. The `register` keyword is required, and cannot be combined with `static`. The register name must be a valid register name for the target platform.

Do not use type qualifiers such as `const` and `volatile`, as the outcome may be contrary to expectations. In particular, when the `const` qualifier is used, the compiler may substitute the variable with its initializer in `asm` statements, which may cause the corresponding operand to appear in a different register.

As with global register variables, it is recommended that you choose a register that is normally saved and restored by function calls on your machine, so that calls to library routines will not clobber it.

The only supported use for this feature is to specify registers for input and output operands when calling Extended `asm` (see [Extended Asm](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html#Extended-Asm)). This may be necessary if the constraints for a particular machine don’t provide sufficient control to select the desired register. To force an operand into a register, create a local variable and specify the register name after the variable’s declaration. Then use the local variable for the `asm` operand and specify any constraint letter that matches the register:

```
register int *p1 asm ("r0") = …;
register int *p2 asm ("r1") = …;
register int *result asm ("r0");
asm ("sysint" : "=r" (result) : "0" (p1), "r" (p2));
```

*Warning:* In the above example, be aware that a register (for example `r0`) can be call-clobbered by subsequent code, including function calls and library calls for arithmetic operators on other variables (for example the initialization of `p2`). In this case, use temporary variables for expressions between the register assignments:

```
int t1 = …;
register int *p1 asm ("r0") = …;
register int *p2 asm ("r1") = t1;
register int *result asm ("r0");
asm ("sysint" : "=r" (result) : "0" (p1), "r" (p2));
```

Defining a register variable does not reserve the register. Other than when invoking the Extended `asm`, the contents of the specified register are not guaranteed. For this reason, the following uses are explicitly *not* supported. If they appear to work, it is only happenstance, and may stop working as intended due to (seemingly) unrelated changes in surrounding code, or even minor changes in the optimization of a future version of gcc:

- Passing parameters to or from Basic `asm`
- Passing parameters to or from Extended `asm` without using input or output operands.
- Passing parameters to or from routines written in assembler (or other languages) using non-standard calling conventions.

Some developers use Local Register Variables in an attempt to improve gcc’s allocation of registers, especially in large functions. In this case the register name is essentially a hint to the register allocator. While in some instances this can generate better code, improvements are subject to the whims of the allocator/optimizers. Since there are no guarantees that your improvements won’t be lost, this usage of Local Register Variables is discouraged.

On the MIPS platform, there is related use for local register variables with slightly different characteristics (see [Defining coprocessor specifics for MIPS targets](http://gcc.gnu.org/onlinedocs/gccint/MIPS-Coprocessors.html#MIPS-Coprocessors) in GNU Compiler Collection (GCC) Internals).

## Size of an Assembly Block

Some targets require that GCC track the size of each instruction used in order to generate correct code. Because the final length of the code produced by an `asm` statement is only known by the assembler, GCC must make an estimate as to how big it will be. It does this by counting the number of instructions in the pattern of the `asm` and multiplying that by the length of the longest instruction supported by that processor. (When working out the number of instructions, it assumes that any occurrence of a newline or of whatever statement separator character is supported by the assembler — typically ‘;’ — indicates the end of an instruction.)

Normally, GCC’s estimate is adequate to ensure that correct code is generated, but it is possible to confuse the compiler if you use pseudo instructions or assembler macros that expand into multiple real instructions, or if you use assembler directives that expand to more space in the object file than is needed for a single instruction. If this happens then the assembler may produce a diagnostic saying that a label is unreachable.