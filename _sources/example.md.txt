# Examples

## Boomerang

```ocaml
# boomerang.tz
parameter unit;
storage unit;
<< ContractAnnot
   { (_, _) | True } ->
   { (ops, _) | amount = 0 && ops = [] ||
      amount <> 0  && (match contract_opt source with
                         | Some c -> ops = [ TransferTokens Unit amount c ]
                         | None -> False) } &
   { _ | False } >>
code
  {
    CDR;
    NIL operation;
    AMOUNT;
    PUSH mutez 0;
    IFCMPEQ
      {
      }
      {
        SOURCE;
        CONTRACT unit;
        ASSERT_SOME;
        AMOUNT;
        UNIT;
        TRANSFER_TOKENS;
        CONS;
      };
    PAIR;
  }
```

The above code, which is the contents of `boomerang.tz` in the
container, is a Michelson program that transfers money amount
`balance` to an account `source`.  The program comes with an
annotation surrounded by `<<` and `>>`.  This annotation, which is
labeled by a constructor `ContractAnnot`, states the following two
properties.

+ The pair `(ops, _)`, which is in the stack at the end of the
program, satisfies `ops = [TransferTokens Unit balance addr]`; this
operation means that this contract will send money amount `balance`
to `addr` with argument `Unit` after this contract finishes.
+ No exceptions are raised from the instructions in this program; this
is expressed by the part `... & { exc | False }`.  There is an
`ASSERT_SOME` instruction in the program that may raise an exception
when the stack top is `None`, but since, from the specification of
Michelson, the account pointed to by `source` should be a
human-operated account, the `CONTRACT unit` should always return
`Some`, so no exception will be raised.

If you verify this program, you will get `VERIFIED`.

## Checksig

```ocaml
# checksig.tz
parameter (pair signature string);
storage (pair address key);
<< ContractAnnot
  { ((sign, data), (addr, pubkey)) |
     match contract_opt addr with
     Some (Contract<string> _) -> True | _ -> False } ->
  { ([op], new_store) |
     (addr, pubkey) = new_store &&
     sig pubkey sign (pack data) &&
     match contract_opt addr with
     | Some c -> op = Transfer data 1 c
     | None -> False }
  & { _ | not (sig pubkey sign (pack data)) } >>
code
  {
    DUP; DUP; DUP;
    DIP { CAR; UNPAIR; DIP { PACK } }; CDDR;
    CHECK_SIGNATURE; ASSERT;

    UNPAIR; CDR; SWAP; CAR;
    CONTRACT string; ASSERT_SOME; SWAP;
    PUSH mutez 1; SWAP;
    TRANSFER_TOKENS;

    NIL operation; SWAP;
    CONS; DIP { CDR };
    PAIR;
  }
```

This program checks the signature given by the parameter is valid and
the contract that the address in storage points to has type `contract
string`; this behavior is described in the post-condition part of
`ContractAnnot`.  The exception part in `ContractAnnot` expresses that
the program can raise two kinds of exceptions: `Error Unit` in
`ASSERT` and `Error 0` in `FAILWITH`.

## Sumseq

```ocaml
# sumseq.tz
parameter (list int);
storage int;
<< Measure sumseq : (list int) -> int where Nil = 0 | Cons h t = (h + (sumseq t)) >>
<< ContractAnnot { arg | True } ->
  { ret | sumseq (first arg) = second ret } &
  { exc | False } (l:list int) >>
code
  {
    CAR;
    << Assume { x | x = l } >>
    DIP { PUSH int 0 };
    << LoopInv { r:s | l = first arg && s + sumseq r = sumseq l } >>
    ITER { ADD };
    NIL operation;
    PAIR;
  }
```

This contract computes the sum of the integers in the list passed as a
parameter.  The `ContractAnnot` annotation uses the function `sumseq`,
which is defined in the earlier `Measure` annotation.  In the `code`
section, the `Assume` annotation is used to specify that `l`, which is
declared to be a ghost variable in the `ContractAnnot`, is the list
passed as a parameter.  `LoopInv` gives the loop-invariant for
`ITER`. `ITER { ADD }` is an instruction that adds the head of the
list to the second `s` from the top of the stack. The loop-invariant
condition `s + sumseq r = sumseq l` expresses that adding `s` to the
sum of list `r` in process equals the sum of the list `l`.


