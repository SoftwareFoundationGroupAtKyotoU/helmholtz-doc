# Developer Document

This document summarizes the implementation issues of Helmholtz.  The implementation follows
the theoretical result~\cite{}, but several extensions and technique is required so that Helmholtz can successfully verify many programs and specifications.

## Overview of Verification Algorithm

Helmholtz verifies code following a *refinement* type system.  As Michelson does typecheck following
the typing rules, we have extended the typing rules with refinements, first-order logic predicates.
Following the refinement type system, given contract annotations, Helmholtz does:

- calculating the post-condition of program code under the given pre-condition;
- trying to fill the gap between the calculated condition and given post-condition; and
- showing the result whether the trial succeeds or not.

Now let's see each step in more detail.

### Calculating a post-condition

The following is one of the refinement typing rules.

$$\Gamma \vdash \{ x_1:\mathtt{int} \rhd x_2:\mathtt{int} \rhd \Upsilon \mid \phi \} \; \mathtt{ADD}
\; \{ x_3:\mathtt{int} \rhd \Upsilon \mid \exists x_1 x_2. \phi \wedge x_1 + x_2 = x_3 \}$$

In the rule, $\Gamma$ ranges over type bindings, $\Upsilon$ ranges over stack bindings, and $\phi$
ranges over first-order logic formulae.  The rule consists of a type binding and two refinement
stack types.  The refinement stack type in the left side of `ADD` expresses the pre-condition of the
instruction, and the right side one expresses the post-condition of the instruction.
So the rule can be read as

> if `ADD` is executed under any stack condition, denoted by $\phi$, then the resulted stack
> satisfies the condition $\exists x_1 x_2. \phi \wedge x_1 + x_2 = x_3$,

which expresses the precise behavior of `ADD` instruction.  The point of our type system is it is
designed so that, given a pre-condition, we can syntactically calculate the post-condition of
contract code.  For instance, given $\{x_1 \rhd x_2 \rhd [] \mid x_1 = 1 \wedge x_2 = 2\}$, which
means the top of stack is 1 and the second from the top is 2, as a pre-condition of `ADD`, we can
have the post-condition $\{x_3 \rhd [] \mid \exists x_1 x_2. x_1 = 1 \wedge x_2 = 2 \wedge x_1 +
x_2 = x_3\}$, which means the top of stack is 3, following by the typing rule.  In this manner,
Helmholtz calculate the post-condition of contract code from a given pre-condition.

### Trying to fill the gap between calculated condition and given condition

There are three points occurring the gap between a calculated condition and a given condition.

- The post-condition of the whole contract code.  In most cases, a calculated post-condition is syntactically different from the desired condition given by ContractAnnot.  For example, in the
  `ADD` example above, given post-condition would be $\{ x_3 \rhd [] \mid x_3 = 3 \}$, a more
  straightforward refinement stack type.
- The pre- and post-condition of `LAMBDA` body.  As you can see in the corresponding typing rule,
  those condition cannot be found out from the pre-condition of `LAMBDA` instruction itself.  So we give those as LambdaAnnot.  For the same reason as the ContractAnnot case, a calculated post-condition will differ from the given post-condition.
- A loop invariant of loop-like instructions, which is also given as a LoopInv annotation.  In this
  case, the calculated post-condition of instruction just before a loop-like instruction and the
  calculated post-condition of the body of the loop-like instruction differ from the given ones.

In the theoretical domain, the typing rule used to fill the gap is RT-Sub.  To apply the rule, Helmholtz
checks if the subtyping relation holds.  By definition of the relation, for example, Helmholtz needs
to know the validity of

$$ \forall \Gamma x_3 . (\exists x_1 x_2. x_1 = 1 \wedge x_2 = 2 \wedge x_1 + x_2 = x_3) \Rightarrow
(x_3 = 3) $$

in the case mentioned in the first item of the last itemization.  To do so, Helmholtz uses the Z3
SMT-solver to check satisfiability of the following formula, called *verification condition*.

$$ \neg (\forall \Gamma x_3 . (\exists x_1 x_2. x_1 = 1 \wedge x_2 = 2 \wedge x_1 + x_2 = x_3)
\Rightarrow (x_3 = 3)) $$

The verification condition is the negation of the formula that we want to know its validity.  So, if
Z3 reports the verification condition is satisfiable, which means there is a counter-example; the
original formula is invalid.  Otherwise, if Z3 reports unsatisfiable, which means there is no
counter-example, the original formula is valid.

### Showing the result

This step is rather simple.  Helmholtz reads a response of Z3.  Then, if all verification conditions are unsatisfiable, Helmholtz reports VERIFIED message, or otherwise, UNVERIFIED message.

## Tezos Specific First-order Theory in Z3

The refinement type system uses a Michelson specialized first-order theory to describe the predicate
formulae.  For example, the theory contains all Michelson types as sorts and various functions and
predicates simulating Michelson instructions.  Z3 is implemented rather rich theory, but of course,
our Tezos specific theory is out of the scope of the native theory in Z3.  Here we explain how our
theory is implemented on Z3.  Note that the method explained here is a well-known method among logic
textbooks.

### Tezos specific functions and predicates

To keep the type system simple, we enrich the logical functions and predicates and just use those in
the post-condition of various instructions.  For example, the following is the typing rule for `SHA256`.

$$ \Gamma \vdash \{ x:\mathtt{bytes} \rhd \Upsilon \mid \phi \} \; \mathtt{SHA256} \; \{
y:\mathtt{bytes} \rhd \Upsilon \mid \exists x. \phi \wedge y = \text{sha256}(x) \}, $$

where $\text{sha256}$ is a logical function respecting `SHA256` behavior, that is, it (ideally)
returns the sha256 hash of a given argument.  To represent the function in Z3, we declare the
function symbol sha256 called an *uninterpreted function* and give axioms for the functions.
Concretely, we give the following code for Z3 input.

``` scheme
(declare-fun sha256 (String) String)
(assert (forall ((x String)) (= (str.len (sha256 x)) 32)))
```

In the first line, the function symbol is declared and, in the second line, we give the axiom that says
the length of the hash is 32.  Note that `bytes` sort is encoded as `String` of Z3 sorts, and `str.len` is a Z3 provided logic function that returns the length of a given string.

Many readers have noticed that the axiom given is insufficient since it says nothing about how the hash
is calculated.  Actually, most logical functions in Helmholtz which are encoded as uninterpreted functions are underspecified because precise axiomatization requires quite complicated formulae,
which lead Z3 execution to diverge.  As a drawback, for instance, the formula `sha256 0x0 =
0x6e340b9cffb37a989ca544e6bb780a2c78901d3fb33738768511a30617afa01d` cannot be validate by Helmholtz,
despite the fact that the SHA-256 hash value of 0x0 is actually the value in the right side of the
equality.  However, we observe that such a lightweight encoding well works because the fact that a
function is applied to some terms itself is more important than what the result of the application
is in many cases.  Moreover, we could manually supply such insufficient facts as the pre-condition
of a contract if we really need it.

#### Measure functions

Measure functions are also encoded as uninterpreted functions.  Thanks to the restricted form of
measure annotations, Z3 input can be easily obtained.  For example, the following is the most
general form of the measure annotation for lists.

``` ocaml
<< Measure f : list T -> T' where [] = e1 | h :: t = e2 >>
```

It is easily imagined that the following Z3 input can be derived from the annotation.

``` scheme
(declare-fun f ((List T)) T')
(assert (= (f []) e1))
(assert (forall ((h T) (t (List T))) (= (f (cons h t)) e2)))
```

In the original work~\cite{}, measures are treated as a part of a type system.  Roughly speaking,
the asserted axioms are systemically (instantiated and) embedded into typing rules.  We find that
measures can be better handled at logic level than a type system because a type system does not need
to concern measures, which now become usual logical functions, and logic can reason about properties
of measures independently from a type system.  Note that the original idea is inherited as the axiom
instantiation described in \secref{}.  So our treatment can be seen as an extension of the original
work.


#### Overloaded functions

Corresponding to the fact that Michelson has overloaded instructions, there are overloaded logical
functions like `pack`.  Z3 has no facility to declare an overloaded function, but an obvious idea to
deal with such functions is to declare functions for every sort, e.g., `pack!int`, `pack!string`,
etc.  One worry is that there are infinite functions for an overloaded function since there are
infinite sorts because of compound data types, but declaring finite ones in Z3 input suffices since
only finite sorts occur in each contract code.  (Of course, a set of declarations varies according
to the contract code.)

### Tezos specific sorts

Although Z3 has a lot of sorts and the facility to declare algebraic datatypes, we still map some
Helmholtz sorts into Z3 sorts, e.g., `nat` is mapped to `Int`.  The problem is that an original sort
is usually a subset of a target sort, and so, naively replacing sorts loses the subset fact.  For
example, $\forall x:\mathtt{nat}. x \ge 0$ in Helmholtz should be valid, but the naively encoded
formula `(forall ((x Int)) (>= x 0))` is invalid in Z3.

To amend the problem, we adopt the method encoding a multi-sorted logic into a one-sorted
logic~\cite{}.  Firstly, we define the *sort predicate* $P_T(x)$ for each sort $T$, which
characterizes the original range of the sort in a target sort.  For example, $P_\mathtt{nat}(x) ::=
x \ge 0$.  Note that, because of compound data types, those definitions would be more complicated
than the first thought, but honestly speaking, those can be defined in first-order predicate logic
and, by similar reason to the overloaded functions, finite sort predicates suffices for each
verification.

Now we amend the lost condition by the sort predicates, that is, $\forall x:T. \phi$ is encoded into
$\forall x:[\![ T ]\!]. P_T(x) \Rightarrow \phi$ and $\exists x:T. \phi$ is encoded into $\exists
x:[\![ T ]\!]. P_T(x) \wedge \phi$, where $[\![ T ]\!]$ denotes the target sort of $T$ (e.g., $[\![
\mathtt{nat} ]\!] = \mathtt{Int}$).  Furthermore, we also amend axioms about co-domain of
uninterpreted functions as $\forall \overline{x_i:T_i}. P_T(f(\bar{x_i}))$ for the function $f$ of
$\overline{T_i} \rightarrow T$ to prevent Z3 from finding an unintended model.

## Implementation Technique

The difficult part of the verification is the second step of the algorithm we have introduced.  It
is undecidable in general since we need a validity check for an undecidable first-order theory.  Here,
we introduce several not perfect but very effective techniques handling logical quantifiers in Helmholtz since those are a large factor making the step undecidable.

### Quantifiers in verification conditions

As we have seen, verification conditions have the following form, which will be undecidable because
$\phi_\text{pre}$ and $\phi_\text{post}$ will involve more quantifiers.

$$ \neg (\forall \Gamma \Upsilon. \phi_\text{pre} \Rightarrow \phi_\text{post}) $$

To resolve the problem, the first observation is that $\phi_\text{post}$ always comes from given
annotations.  So, we design the annotation language so that it does not involve quantifies and make
$\phi_\text{post}$ quantifier-free, denoted by $\phi_\text{post}^\text{QF}$.  The second observation
is that the calculated condition can be the form of $\exists \bar{x}. \phi^\text{QF}$ (except the typing
rules for `LAMBDA` and `APPLY`).  So, now the verification condition can be equivalent to

$$ \exists \Gamma \Upsilon \bar{x}. \phi_\text{pre}^\text{QF} \wedge \neg \phi_\text{post}^\text{QF}, $$

which is much closer to a decidable fragment of the theory.

### Heuristic instantiation of universally quantified axioms

Another source of quantifiers is the axioms of the theory.  As we have already seen, various axioms
involve the universal quantifier.  Naively put such axiom in Z3 input, Z3 soon diverges.  Especially,
verifying contract code expected UNVERIFIED tends to diverge because Z3 needs to find a
counter-example for the given specification, which involves finding a model of uninterpreted
functions.  Such a task is sometimes too hard if the functions are constrained by universally
quantified axioms.

To avoid difficulty, we give instantiated axioms for Z3 input instead of original universally
quantified ones.  Of course, instantiated ones are less precise than the original ones.  For
example, consider the axioms for a list measure.  We naively put the first axiom, which is the case
for the nil list, because it is not quantified.  Regarding the second axiom, which is the case for a
cons list, we do not give the axiom to Z3 input.  Instead, if we find a term of the form `(cons e_h
e_t)` in formulae dealt with, we give the axiom obtained by replacing `h` and `t` with `e_h` and
`e_t`, respectively, in `(= (f (cons h t)) e2)`, which is one instance of the second axiom.  What
terms are chosen for instantiation is determined by heuristics.  More instances would lead
Helmholtz to verify more contracts but take more time.
