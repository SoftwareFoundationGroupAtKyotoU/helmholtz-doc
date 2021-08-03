# Helmholtz

Helmholtz is a static verification tool for a stack-based programming language
[Michelson](https://tezos.gitlab.io/whitedoc/michelson.html), a smart contract language used in
[Tezos](https://tezos.gitlab.io/) blockchain protocol.  It verifies that a Michelson program
satisfies a user-written formal specification.

Helmholtz accepts a [Michelson](https://tezos.gitlab.io/whitedoc/michelson.html) program annotated
with its formal specification and hints (e.g., loop invariant) used by Helmholtz.  An annotation is
surrounded by ``<<`` and ``>>``.

Helmholtz works as follows.
- Helmholtz strips and preserves the annotations surrounded by ``<<`` and ``>>`` and typechecks the
stripped code using ``tezos-client typecheck``; the simple type checking is conducted in this step.
- After typechecking, it generates verification conditions using the preserved annotations.
- Then, Helmholtz discharges the conditions with `z3` and outputs `VERIFIED` or `UNVERIFIED`.

## Specification of Assertion Language

### Refinement Stack Type

    RTYPE ::= "{" STACK "|" EXP "}"
    STACK ::= PATTERN |  PATTERN ":" STACK

Refinement stack types are broadly used expressions to describe a state of stack values.  A
refinement stack type consists of `STACK`, a stack pattern that binds variables to a matched stack
values, and `EXP`, a predicate expression that describes a condition for the bound variables.

For instance, `{ x | True }` denotes any singleton stack because the stack pattern `x` matches a
stack of exactly one length and the predicate expression `True` gives no condition on `x`.  In a
similar manner, `{ x : y | True }` denotes a stack of two length, `{ x : y : z | True }` denotes a
stack of three length, etc.  A predicate expression can be any Boolean expression.  So, for
instance, `{ x | x > 0 }` denotes a singleton stack of positive integers.

It sometimes happens that we are just interested in some top values of a stack.  In such a case, we
can use *any pattern* `_` in the bottom position of a stack pattern, which denotes a stack of any
length (including the nil stack).  So, `{ _ | True }` denotes any stack, `{ x : _ | True }` denotes
a stack of more than zero length, etc.  Note that `{ x : _ : y | True }` denotes a stack of exactly
three length since any pattern is not placed in the bottom position and so it just matches any
*value* in the middle of the stack.

In general, we can use arbitrary patterns to match values in a stack.  For instance, `{ (x, y) |
True }` denotes a singleton stack of a pair.  Roughly speaking, `{ p1 : ... : pn | e }` is
interpreted as `{ x1 : ... : xn | match x1, ..., xn with p1, ..., pn -> e | _ -> False }` (in
precise, there are a few differences, e.g., when `pn = _`, though).  So, `{ [x] | True }` denotes a
singleton stack of a singleton list (it is equivalent to `{ y | match y with [x] -> True | _ ->
False }`.

#### Typechecking for Refinement Stack Types
Given refinement stack types are typechecked to be a valid expression.  For example, when `{ x | x >
0 }` is placed in a code point in which a stack type is `int : []` (as you know, Michelson code is
statically typed, so we know a stack type of each code point), a type of `x` is inferred as `int`
and the refinement stack type is accepted as a valid expression.  Contrary, when it is placed in a
code point expecting `bool : []`, Helmholtz reports a type error because a type of `x` is inferred
as `bool` and the predicate expression is ill-typed since `0` is not a Boolean value.  Length
mismatch is also reported as a type error like the Michelson type system does.

### Annotations

    ANNOTATION ::=
      | "Assert" RTYPE
      | "LoopInv" RTYPE
      | "Assume" RTYPE
      | "LambdaAnnot" RTYPE "->" RTYPE "&" RTYPE [ "(" (VAR ":" SORT)+ ")" ]
      | "ContractAnnot" RTYPE "->" RTYPE "&" RTYPE [ "(" (VAR ":" SORT)+ ")" ]
      | "Measure" VAR ":" SORT "->" SORT "where" "[" "]" "=" EXP "|" VAR "::" VAR "=" EXP
      | "Measure" VAR ":" SORT "->" SORT "where" "EmptySet" "=" EXP "|" "Add" VAR VAR "=" EXP
      | "Measure" VAR ":" SORT "->" SORT "where" "EmptyMap" "=" EXP "|" "Bind" VAR VAR VAR "=" EXP

An annotation is a root object of annotations for Helmholtz.  It gives several operations, hints,
etc. to Helmholtz.  There are six kinds of annotations in Helmholtz: `ContractAnnot`, `LambdaAnnot`,
`LoopInv`, `Assert`, `Assume`, and `Measure`.  In those, `ContractAnnot` is mandatory for contract
code, and `LambdaAnnot` and `LoopInv` is mandatory for the `LAMBDA` instruction and loop
instructions (`LOOP`, `ITER`, etc.), respectively.

#### Contract Annotations

A contract annotation gives a specification of contract.  It is placed just before the code section
and causes Helmholtz to check if the contract satisfies the given specification.  A contract
annotation has the form `ContractAnnot rtype_pre -> rtype_post & rtype_abpost vars`, where `vars` is
optional.

- `rtype_pre` describes a supposed state of an initial stack.  So it is a refinement stack type for
  a stack type `(pair parameter_ty storage_ty) : []`.  The stack pattern in `rtype_pre` is
  restricted to the form of `(p1, p2)` or `_` (simple variable pattern `x` is prohibit in other
  words) because of technical reason.  A typical `rtype_pre` is `{ _ | True }`, which means a
  contract can accept any parameter and storage value.  It is also reasonable to give an assumption
  for a storage value, since the storage is controlled by a contract, like `{ (_, addr) | match
  contract_opt addr with Some (Contract<string> _) -> True | None -> False }` which supposes the
  destination of an address stored in the storage must exist and of which parameter type is
  `string`.
- `rtype_post` describes a desirable state of an final stack.  So it is a refinement stack type for
  a stack type `(pair (list operation) storage_ty) : []`.  Helmholtz checks if this condition holds
  under the assumption described by `rtype_pre` and by analyzing the contract code.
- `rtype_abpost` describes a exceptional value condition.  It is a refinement stack type for the
  stack type `exception : []`, where `exception` is a special data type consists of constructors
  `Error v` for a `FAILWITH` exception and `Overflow` for the overflow exception.  A typical
  `rtype_abpost` is `{ _ | False }`, which means no exception happens since no value can satisfy the
  `False` condition.
- Optional `vars` declares variables that can be used in annotations occurring in contract code
  (except the body of `LAMBDA` instruction).  So those cannot occur in `rtype_*`.

##### Special treatment of the scope of pre-condition stack variables.

Basically scope of the variables in the stack pattern in a refinement stack type is in its predicate
expression.  The only exception is ones in `rtype_pre`.  Its scope also involves predicate
expressions in `rtype_post`, `rtype_abpost`, and annotations occurring in contract code (except the
body of `LAMBDA` instruction).

#### Lambda Annotations

A lambda annotation is a similar annotation to contract annotations but for `LAMBDA` instruction.
So it must be placed just before the `LAMBDA` instruction and causes Helmholtz to check if a lambda
pushed by the instruction satisfies the given specification.  A lambda annotation has the form
`LambdaAnnot rtype_pre -> rtype_post & rtype_abpost vars` and each component has similar meaning to
ones of a contract annotation.

#### Loop Invariant

A loop invariant annotation gives a loop invariant for loop-like instructions: `ITER`, `LOOP`, and
`LOOP_LEFT` (`MAP` is not unsupported by Helmholtz yet); and so, it must be placed just before those
instruction.  A loop invariant annotation has the form `LoopInv rtype`, where `rtype` gives a loop
invariant.  A loop invariant is a desired condition for a stack just before a loop iteration.

#### Assert

An assert annotation lets Helmholtz check a stack condition at the given code point.  An assertion
annotation has the form `Assert rtype`, where `rtype` gives a stack condition being verified.

#### Assume

An assume annotation adds a fact (hypothesis) about a stack condition at the given code point.  An
assertion has the form `Assume rtype`, where `rtype` gives a fact for a stack condition.  A user can
give any fact, and Helmholtz believes that the fact is correct, that means, Helmholtz never checks
if the fact hold.  So, this annotation must be carefully used because, if you gives an incorrect
fact, a verification result becomes nonsensical.

#### Measure

### Patterns and Expressions

    VAR ::= [a-z][a-z A-Z 0-9 _ ']*
    PATTERN ::=
      | VAR
      | CONSTRUCTOR ("<" SORT ">")? (PATTERN)*
      | PATTERN "," PATTERN
      | PATTERN "::" PATTERN
      | "[" "]"
      | "[" PATTERN (";" PATTERN)* "]"
      | "_"
      | "(" PATTERN ")"
    EXP ::=
      | VAR
      | NUMBER
      | STRING
      | BYTES
      | UOP EXP
      | EXP BOP EXP
      | CONSTRUCTOR (EXP)*
      | FUNCTION (EXP)*
      | EXP "." ACCESSER
      | "if" EXP "then" EXP "else" EXP
      | EXP "," EXP
      | EXP ":" SORT
      | VAR ":>" RTYPE "->" RTYPE "&" RTYPE
      | "[" "]"
      | "[" EXP (";" EXP)* "]"
      | "match" EXP "with" PATTERN-MATCHING
      | "(" EXP ")"
    PATTERN-MATCHING ::= ("|")? PATTERN "->" EXP ("|" PATTERN "->" EXP)*
    OP ::= "+" | "-" | "*" | "/" | "mod" | "<" | ">" | "<=" | ">=" | "=" | "<>" | "&&" | "||" | "::" | "^"
    UOP ::= "-" | "!"
    ACCESSER ::= "first" | "second"

Patterns and expressions are similar to ones of a functional programming language (basically we
follow OCaml).  However there are notable difference from a general programming language because the
expressions are designed for first-order logic:

- Constructors and functions must be fully applied.
- There is no facility to define new data types and functions.  (Measure annotation could be an
  exception, but the ability is restricted.)  So, only built-in constructors and (measure) functions
  can be used.

#### Constructors

- `True` : `bool` True value of Boolean type.
- `False` : `bool` False value of Boolean type.
- `Unit` : `unit` The unique value of the unit type.
- `Nil` : `list 'a` The empty list.
- `Cons` : `'a -> list 'a -> list 'a` Prepend an element to a list.
- `Pair` : `'a -> 'b -> pair 'a 'b` Make the pair of two elements.
- `None` : `option 'a` Absent value of the option type.
- `Some` : `'a -> option 'a` Present value of the option type.
- `Left` : `'a -> or 'a 'b` Left alternative of the or type.
- `Right` : `'a -> or 'b 'a` Right alternative of the or type.
- `Contract` : `address -> contract 'a` A value of the contract type.  This constructor can be used
only in a pattern, that is, used for deconstructing a contract object.  Use `contract_opt` for
construction.
- `SetDelegate` : `option key -> operation` Delegate operation
- `TransferTokens` : `'a -> mutez -> contract 'a -> operation` Transaction operation.  There is the
short alias `Transfer`.
- `CreateContract` : `option address -> mutez -> 'a -> address -> operation` Contract origination
  operation.
- `Error` : `'a -> exception` Represents an exception raised by `FAILWITH` instruction.
- `Overflow` : `exception` Represents an overflow exception.

##### Special form in a pattern

In a pattern, `Contract`, `TransferTokens`, `CreateContract`, and `Error` can be optionally followed
by a type parameter of the form `<ty>` to give the specific type `ty` for those unique type
variable.  For instance, the pattern `Contract <nat> x` is matched for a value of the type `contract
nat`.

Extra attention would be required for `TransferTokens`, `CreateContract`, and `Error` in a pattern.
That is because, for example, a value of the type `exception` can be `Error <ty> v` for any `ty`
(and `v` of `ty`).  It implies the only way to make match expression for `exception` values be
exhaustive is to include *any pattern* (default) case.  (Currently we have no facility to match *any
type parameter*.)

#### Functions

`Function` is one of the builtin functions and user defined measures.  The following is a list of
the builtin functions and those descriptions.

- `not : bool -> bool`: `not b` negates the Boolean value `b`.
- `get_str_opt : string -> nat -> option string`: `get_str_opt s i` returns `i`-th character of `s`
as the one-length string.
- `sub_str_opt : string -> nat -> nat -> option string`: `sub_str_opt s i l` returns the substring of
`s` from `i`-th with the length of `l`.
- `len_str : string -> nat`: `len_str s` returns the length of the string `s`.
- `concat_str : string -> string -> string`: `concat_str s1 s2` concatenates the two strings `s1`
and `s2`.  This is equivalent to `s1 ^ s2`.
- `get_bytes_opt : bytes -> int -> bytes`
- `sub_bytes_opt : bytes -> int -> int -> bytes`
- `len_bytes : bytes -> int`
- `concat_bytes : bytes -> bytes -> bytes`
- `first : pair 'a 'b -> 'a`
- `second : pair 'a 'b -> 'b`
- `pack : 'a -> bytes`: A value with this constructor corresponds to a value created by the
instruction `PACK` in Michelson.  It is expressed by a constructor because it needs type
information.
- `unpack_opt : bytes -> option 'a`
- `find_opt : 'a -> map 'a 'b -> option 'b`: `find_opt k m` looks for an entry associated with the
key `k` from the map `m`.  It returns `Some v` if `v` associated with `k` in `m`, or otherwise, it
returns `None`.
- `update : 'a -> option 'b -> map 'a 'b -> map 'a 'b`: `update k (Some v) m` updates the value
associated with `k` in `m` to `v`; `update k None m` deletes the value associated with `k` from `m`.
- `empty_map : map 'a 'b`
- `mem : 'a -> set 'a -> bool`
- `add : 'a -> set 'a -> set 'a`
- `remove : 'a -> set 'a -> set 'a`
- `empty_set : set 'a`
- `source : address`: Corresponding to `SOURCE` in Michelson.
- `sender : address`: Corresponding to `SENDER` in Michelson.
- `self : contract parameter_ty`: Corresponding to `SELF` in Michelson.
- `self_addr : address`
- `now : timestamp`: Corresponding to `NOW` in Michelson.
- `balance : mutez`: Corresponding to `BALANCE` in Michelson.
- `amount : mutez`: Corresponding to `AMOUNT` in Michelson.
- `chain_id : chain_id`
- `level : nat`
- `total_voting_power : nat`
- `contract_opt : address -> option (contract 'a)`
- `implicit_account : key_hash -> contract unit`
- `call : fun 'a 'b -> 'a -> 'b -> bool`: `call f a b`, where `f` returns `true` if the application
 of function `f` created by `LAMBDA` to argument `a` terminates and evaluates to `b`; `false`
 otherwise.
- `hash : key -> key_hash`: Corresponding to `HASH` in Michelson.
- `blake2b : bytes -> bytes`: Corresponding to `BLAKE2B` in Michelson.
- `keccak : bytes -> bytes`
- `sha256 : bytes -> bytes`: Corresponding to `SHA256` in Michelson.
- `sha512 : bytes -> bytes`: Corresponding to `SHA512` in Michelson.
- `sha3 : bytes -> bytes`
- `sig : key -> signature -> bytes -> bool`: Corresponding to `CHECK_SIGNATURE` in Michelson.

### Annotations

In the following explanations of annotations, `rtype` represents a
refinement type `{ stack | exp }` a pattern that maches a stack; `exp`
is an expression of type `bool`.  It represents a `stack` in which
`exp` evaluates to `true`.

There are 6 types of annotations below.

- `ContractAnnot rtype1 -> rtype2 & rtype3 vars`

- This annotation gives the specification of a contract.  If this contract is executed with an initial stack that satisfies the pre-condition `rtype1`, and if it finishes its execution without exception, then the resulting stack satisfies the post-condition `rtype2`; if an exception is raised, then the exception value satisfies `rtype3`.

- `rtype1` is a pre-condition for the stack (=`[pair parameter_ty storage_ty]`) when the program starts.
- `rtype2` is a post-condition for the stack (=`[pair (list operation) storage_ty]`) when the program ends.
- `rtype3` is a refinement type for the value the exception the program may throw has.
- It is possible to declare ghost variables in `vars` that can be used in annotation inside the program.

- The ghost variables can be used in the annotations in `code`
section.  They cannot be used in `rtype1`, `rtype2`, and
`rtype3`.

- A `ContractAnnot` annotation must be placed just before a `code` section.

- `LambdaAnnot rtype1 -> rtype2 & rtype3 tvars`

- This annotation gives the specification of a function that is
created by an instruction `LAMBDA`.  A `LAMBDA` instruction
associated with an annotation `LambdaAnnot rtype1 -> rtype2 &
rtype3 tvars` should behave as if it is a contract annotated with
`ContractAnnot rtype1 -> rtype2 & rtype3`.
- A `LambdaAnnot` annotation must be placed just before `LAMBDA`.

- `Assert rtype`

- It asserts that the stack at the program point where this
annotation is placed satisfies `rtype`.
- An assertion is checked by Helmholtz; if it is not verified, then
the result of Helmholtz will be `UNVERIFIED`.

- `Assume rtype`

- It assumes that the stack at the program point where this
annotation is placed satisfies `rtype`.
- Helmholtz assumes that a correct assumption is given.  It is
user's responsibility to make sure that the assumption is correct.
If a wrong assumption is given, the verification result may not be
reliable.

- `LoopInv rtype`

- This annotation declares a loop invariant.  It is placed just
before `LOOP` and `ITER` and specifies that the stack at the
beginning of each iteration satisfies `rtype`.
- A `LoopInv` annotation must be placed just before `LOOP`, `ITER`

- `MAP`, `LOOP_LEFT` are not yet supported

- `Measure`

- This annotation defines a (recursive) function over a list, a set,
or a map that can be used in annotations.
- `Measure` annotations should be placed before `ContractAnnot`.


### Details

- `Key`, `Address`, `Signature` are not defined as constructors.  This is because we don't want to deconstruct the `key`, `address`, and `signature` values into a string or bytes.
- The `str` in `Timestamp str` accepts an RFC3339-compliant string.
- The current annotation language does not distinguish between `int` and `nat`, `mutez`, and `timestamp` at the type level.
- Operator precedence follows the convention of OCaml.
- `LOOP_LEFT`, `APPLY`, (`LSL`, `LSR`, `AND`, `OR`, `XOR`, `NOT` as bit operations), `MAP`, (`SIZE` for map, set, and list), `CHAIN_ID`, and deprecated instructions are not yet supported.
- Some relations between constants are not inferred automatically. For examples, despite the fact that `sha256 0x0 = 0x6e340b9cffb37a989ca544e6bb780a2c78901d3fb33738768511a30617afa01d` is true, Helmholtz will not verify this. If you need such properties, use `Assume`.
- When ITERate map or set, Helmholtz can not use the condition about the order of the iteration.
- Our source codes of Helmholtz is in `/home/opam/ReFX/src/proto_005_PsBabyM1/lib_refx` in the container.

### Q&A

- Error `misaligned expression` is output
- It is not an error output by Helmholtz, but an error by indent-check by Michelson `tezos-client typecheck`. For the rules of indentation, see [here](https://tezos.gitlab.io/whitedoc/micheline.html).
- Error `MenhirBasics.Error` is output
- This is an syntax error output by Helmholtz. Please check the annotations you give.

