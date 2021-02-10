# Homework 3: div, mul, and, or, let, case

## `*` and `/`

Extend the interpreter and the compiler to support the `*` and `/`
operators. These are binary operators that perform division and multiplication,
respectively.

In the interpreter, these should be implemented just like `+` and `-`. In the
compiler, things get a little trickier:

- x86-64's `idiv` and `imul` instructions, which do signed multiplication and
  division, are a bit unusual. Both only take one argument, and divide (or
  multiply) `rax` by that argument.
- `div` looks at `rdx` for sign information. Before calling the `div`
  instruction, you should call `cqo` to sign-extend `rax` into `rdx`.
- With `add` and `sub`, we were able to ignore our encoding of integers because
  multiplication distributes over addition. For `mul` and `div`, that won't be
  the case. If you need to shift an integer right, make sure to use `sar`
  instead of `shr` to keep the sign consistent.

Recall that each 64-bit register (e.g. `rax`) can store an integer ranging from 
`-(2**63)` to `(2**63)-1` (both inclusively). When two such integers are
multiplied, the result will range from `-2**126 +2**63` to `2**126`, which 
would fit not in a 64-bit register but an 128-bit one. One can simulate an 
128-bit register with two 64-bit registers. This is why x86_64 assembly puts 
the "outputs" of `imul` in `rdx` and `rax`. Conversely, the dividend for `idiv`
is 128-bit and stored in `rdx` and `rax`.

You may use `sar` and `sal` to cast our 62-bit lisp numbers to and from int64s,
and `cqo` for casts between int64 and int128. Of course, casting from a larger 
integer set to a smaller one may fail (e.g. from `2**63` to a lisp number). You 
are __not__ required to handle these cases in this assignment.


Here is a summary of `cqo`, `sal` and `sar`
(Source: https://www.felixcloutier.com/x86/index.html):

- __`CQO`__: an instruction. `RDX:RAX` := sign-extend of `RAX`
- __`SAL r/m64, imm8`__: multiply `r/m64` by `2`, `imm8` times
- __`SAR r/m64, imm8`__: signed divide `r/m64` by `2`, `imm8` times



## `and` and `or`

Implement `and` and `or`, binary operations that take operands of any type and
return one of them as described below. As usual, only `false` is considered falsey.

```
(or <non-falsey-value-1> <value-2>) --> <non-falsey-value-1>
(or false <value-2>) --> <value-2>

(and <non-falsey-value-1> <value-2>) --> <value-2>
(and false <value-2>) -> false
```
  
Both `and` and `or` should *short-circuit*: if they are returning their first
argument, they should not evaluate their second argument. It might be useful to
write a test case that will fail if they do not short-circuit, perhaps using
your new `/` operator with a `0` argument. For this reason, they must *not*
be implemented as binary primitives, since the arguments of binary primitives
are *eagerly* evaluated before `compile_primitive` or `interp_binary_primitive`
are called.

## Multiple bindings in `let`

**Note:** We'll be implementing a `let` construct for binding variables in class
on Thursday, 2/11. This section is best left until after that lecture.

The `let` form we implement in class binds exactly one variable. The
generalized form binds multiple variables, e.g.

```
(let ((x 2)
      (y 3))
  (+ x y))
```

These variables should be bound *simultaneously*: none of the definitions should
see any of the other bindings. This means that you cannot implement this
generalized form as multiple nested let bindings. It may be useful to think of a
test case that will have different behavior if `let` is implemented in this way!

Implement this generalized let form in your interpreter and in your
compiler. We've provided a `get_bindings` helper function in `lib/util.ml` that
may be useful (import the module with `open Util`).

## The `case` form

Common Lisp's `case` expression, like C's `switch` and OCaml's pattern-matching,
compares a single expression against a number of values:

```
(case 4
  (1 1)
  (2 4)
  (3 9)
  (4 16)
  (5 25)) => 16
```


You'll implement a limited form of this expression:
- The argument must be an expression that evaluates to an integer
- The left-hand side of each case must be a *literal* integer
- There must be at least one case
- The last case in the expression is the default case

In your interpreter, you can implement support for this expression however you'd like.

In your compiler, implement support for `case` expressions via [*branch
tables*](https://en.wikipedia.org/wiki/Branch_table). A branch table (sometimes
called a *jump table*) avoids the need for many comparisons and jumps. For
instance, imagine converting the case expression above to `if` expressions:

```
(if (= 4 1)
  1
  (if (= 4 2)
    4
    (if (= 4 3)
      9
      (if (= 4 4)
        16
        25))))
```

If we compiled this code and ran it, the processor would end up needing to
execute 4 `cmp` instructions and 4 `jmp` instructions. If we added more cases,
the worst case scenario gets worse; execution time will be linear in the number
of cases.

For 4 or 5 cases, this isn't usually too terrible. But for some applications,
it's really important to be able to do this kind of switching in constant time
in the number of cases. Branch tables let us do that.


### Building a branch table

A branch table is a chunk of assembly directives that, rather than specifying
instructions, just contain the addresses of labels. It looks like this:

```
branch_table:
   dq case_label_1
   dq case_label_2
   dq case_label_3
   ...
   dq case_label_n
```

(The `dq` instruction puts data into the executable. The `q` is for `quadword`,
because label addresses are 8 bytes.)

The first label in the branch table should be the label of the smallest (not
necessarily first!) case. The last label should be the label of the largest (not
necessarily last!) case. There should be a label for every value in between the
minimum and the maximum; this means you might have more labels than cases. For
the "holes" between cases, you should use the label of the last (not necessarily
largest!) case, since this is the default.

After the branch table, you should emit code for each case's expression, labeled
with the correct label. Like the "then" case of an `if`, each case should jump
to the same "continue" label.

### Using a branch table

To use the branch table, you'll need to:

- Compare the argument to the minimum and maximum cases. If it's outside those
  bounds (i.e., less than the minimum case or greater than the maximum case),
  jump to the default label.
- Compute the offset of the label for your argument. For an argument `x`, this
  is `(x - min_case) * 8`. You need to multiply by 8 because our label addresses
  take up 8 bytes.
- Load the label for the branch table into a register. You should use `LeaLabel`
  for this.
- Jump to the computed label. You should use `ComputedJmp` with a `MemOffset`
  argument for this.

### Useful functions

A few functions you may find useful for implementing `case`:

-   `List.assoc`, `List.assoc_opt` for using `('a * 'b) list`s somewhat like maps.
    -   See the "Association lists" section of [OCaml's List docs](https://caml.inria.fr/pub/docs/manual-ocaml/libref/List.html).
-   `get_cases`, which parses a list of expressions into a list of (number, expression) pairs.
    -   Defined in `lib/util.ml`, which can be opened with `open Util`.
-   `List.range lo hi`, which produces a list of numbers from `lo` to `hi`, inclusive.
    -   Added to the `List` module in `lib/util.ml`, which can be opened with `open Util`.

## Grammar
The grammar your new interpreter and compiler should support is as follows:
```diff
<expr> ::= <num>
         | <char>
         | <id>
         | true
         | false
         | (<un_prim> <expr>)
         | (<bin_prim> <expr> <expr>)
         | (if <expr> <expr> <expr>)
+        | (and <expr> <expr>)
+        | (or <expr> <expr>)
+        | (let ((<id> <expr>) ...) <expr>)
+        | (case <expr> (<num> <expr>) (<num> <expr>) ...)
         

<un_prim> ::= add1
            | sub1
            | zero?
            | num?
            | not

<bin_prim> ::= +
             | -
             | =
             | <
+            | /
+            | *
```
