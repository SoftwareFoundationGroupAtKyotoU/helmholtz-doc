# Instruction support status

## Summary

- `MAP` instruction is not supported.
- Bitwise operation is not supported.
- Types and related instructions of `never`, `ticket`, `bls12_381_g1`, `bls12_381_g2`,
`bls12_381_fr`, `sapling_transaction`, and `sapling_state` are not supported.

## Detailed status

o: supported instruction

| | Instruction             | typing
|-| ----------------------- | ------
|o| ABS                     | int : 'S -> nat : 'S
|o| ADD                     | int : int : 'S -> int : 'S
|o|                         | int : nat : 'S -> int : 'S
|o|                         | nat : int : 'S -> int : 'S
|o|                         | nat : nat : 'S -> nat : 'S
|o|                         | timestamp : int : 'S -> timestamp : 'S
|o|                         | int : timestamp : 'S -> timestamp : 'S
|o|                         | mutez : mutez : 'S -> mutez : 'S
| |                         | bls12_381_g1 : bls12_381_g1 : 'S -> bls12_381_g1 : 'S
| |                         | bls12_381_g2 : bls12_381_g2 : 'S -> bls12_381_g2 : 'S
| |                         | bls12_381_fr : bls12_381_fr : 'S -> bls12_381_fr : 'S
|o| ADDRESS                 | contract _ :: 'S -> address : 'S
|o| AMOUNT                  | 'S -> mutez : 'S
|o| AND                     | bool : bool : 'S -> bool : 'S
| |                         | nat : nat : 'S -> nat : 'S
| |                         | int : nat : 'S -> nat : 'S
|o| APPLY                   | 'a : lambda (pair 'a 'b) 'c : 'C -> lambda 'b 'c : 'C
|o| BALANCE                 | 'S -> mutez : 'S
|o| BLAKE2B                 | bytes : 'S -> bytes : 'S
|o| CAR                     | pair 'a _ : 'S -> 'a : 'S
|o| CAST 'b                 | 'a : 'S -> 'b : 'S <br /> iff 'a = 'b
|o| CDR                     | pair _ 'b : 'S -> 'b : 'S
|o| CHAIN_ID                | 'S -> chain_id : 'S
|o| CHECK_SIGNATURE         | key : signature : bytes : 'S -> bool : 'S
|o| COMPARE                 | unit : unit : 'S -> int : 'S
| |                         | never : never : 'S -> int : 'S
|o|                         | bool : bool : 'S -> int : 'S
|o|                         | int : int : 'S -> int : 'S
|o|                         | nat : nat : 'S -> int : 'S
|o[^compare]|                         | string : string : 'S -> int : 'S
|o|                         | pair 'a 'b : pair 'a 'b : 'S -> int : 'S
|o|                         | option 'a : option 'a : 'S -> int : 'S
|o|                         | or 'a 'b : or 'a 'b : 'S -> int : 'S
|o|                         | timestamp : timestamp : 'S -> int : 'S
|o|                         | mutez : mutez : 'S -> int : 'S
|o[^compare]|                         | chain_id : chain_id : 'S -> int : 'S
|o[^compare]|                         | bytes : bytes : 'S -> int : 'S
|o[^compare]|                         | key_hash : key_hash : 'S -> int : 'S
|o[^compare]|                         | key : key : 'S -> int : 'S
|o[^compare]|                         | signature : signature : 'S -> int : 'S
|o[^compare]|                         | address : address : 'S -> int : 'S
|o| CONCAT                  | string : string : 'S -> string : 'S
|o|                         | string list : 'S -> string : 'S
|o|                         | bytes : bytes : 'S -> bytes : 'S
|o|                         | bytes list : 'S -> bytes : 'S
|o| CONS                    | 'a : list 'a : 'S -> list 'a : 'S
|o| CONTRACT 'p             | address : 'S -> option (contract 'p) : 'S
|o[^create_contract]| CREATE_CONTRACT <br /> { storage 'g ; parameter 'p ; code ... } | option key_hash : mutez : 'g : 'S -> operation : address : 'S
|o| DIG n                   | 'a{1} : ... : 'a{n} : 'b : 'A -> 'b : 'a{1} : ... : 'a{n} : 'A
|o| DIP code                | 'b : 'A -> 'b : 'C <br /> iff   code :: [ 'A -> 'C ]
|o| DIP n code              | 'a{1} : ... : 'a{n} : 'A -> 'a{1} : ... : 'a{n} : 'B <br /> iff   code :: [ 'A -> 'B ]
|o| DROP                    | _ : 'A -> 'A
|o| DROP n                  | 'a{1} : ... : 'a{n} : 'A -> 'A
|o| DUG n                   | 'b : 'a{1} : ... : 'a{n} : 'A -> 'a{1} : ... : 'a{n} : 'b : 'A
|o| DUP                     | 'a : 'A -> 'a : 'a : 'A
|o| DUP n                   | 'a{1} : ... : 'a{n} : 'A -> 'a{n} : 'a{1} : ... : 'a{n} : 'A
|o| EDIV                    | int : int : 'S -> option (pair int nat) : 'S
|o|                         | int : nat : 'S -> option (pair int nat) : 'S
|o|                         | nat : int : 'S -> option (pair int nat) : 'S
|o|                         | nat : nat : 'S -> option (pair nat nat) : 'S
|o|                         | mutez : nat : 'S -> option (pair mutez mutez) : 'S
|o|                         | mutez : mutez : 'S -> option (pair nat mutez) : 'S
|o| EMPTY_BIG_MAP 'key 'val | 'S -> big_map 'key 'val : 'S
|o| EMPTY_MAP 'key 'val     | 'S -> map 'key 'val : 'S
|o| EMPTY_SET 'elt          | 'S -> set 'elt : 'S
|o| EQ                      | int : 'S -> bool : 'S
|o| EXEC                    | 'a : lambda 'a 'b : 'C -> 'b : 'C
|o| FAILWITH                | 'a : \_ -> \_
|o| GE                      | int : 'S -> bool : 'S
|o| GET                     | 'key : map 'key 'val : 'S -> option 'val : 'S
|o|                         | 'key : big_map 'key 'val : 'S -> option 'val : 'S
|o| GET 0                   | 'a : 'S -> 'a : 'S
|o| GET (2k)                | pair 'a{0} ... 'a{k-1} 'a{k} : 'S -> 'a{k} : 'S
|o| GET (2k+1)              | pair 'a{0} ... 'a{k} 'a{k+1} : 'S -> 'a{k} : 'S
|o| GET_AND_UPDATE          | 'key : option 'val : map 'key 'val : 'S -> option 'val : map 'key 'val : 'S
|o|                         | 'key : option 'val : big_map 'key 'val : 'S -> option 'val : big_map 'key 'val : 'S
|o| GT                      | int : 'S -> bool : 'S
|o| HASH_KEY                | key : 'S -> key_hash : 'S
|o| IF bt bf                | bool : 'A -> 'B <br /> iff   bt :: [ 'A -> 'B ] <br /> bf :: [ 'A -> 'B ]
|o| IF_CONS bt bf           | list 'a : 'A -> 'B <br /> iff   bt :: [ 'a : list 'a : 'A -> 'B] <br /> bf :: [ 'A -> 'B]
|o| IF_LEFT bt bf           | or 'a 'b : 'A -> 'B <br /> iff   bt :: [ 'a : 'A -> 'B] <br /> bf :: [ 'b : 'A -> 'B]
|o| IF_NONE bt bf           | option 'a : 'A -> 'B <br /> iff   bt :: [ 'A -> 'B] <br /> bf :: [ 'a : 'A -> 'B]
|o| IMPLICIT_ACCOUNT        | key_hash : 'S -> contract unit : 'S
|o| INT                     | nat : 'S -> int : 'S
| |                         | bls12_381_fr : 'S -> int : 'S
|o| ISNAT                   | int : 'S -> option nat : 'S
|o| ITER body               | (set 'elt) : 'A -> 'A <br /> iff body :: [ 'elt : 'A -> 'A ]
|o|                         | (map 'elt 'val) : 'A -> 'A <br /> iff   body :: [ (pair 'elt 'val : 'A) -> 'A ]
|o|                         | (list 'elt) : 'A -> 'A <br /> iff body :: [ 'elt : 'A -> 'A ]
| | JOIN_TICKETS            | (pair (ticket 'a) (ticket 'a)) : 'S -> option (ticket 'a) : 'S
|o| KECCAK                  | bytes : 'S -> bytes : 'S
|o| LAMBDA 'a 'b code       | 'A -> (lambda 'a 'b) : 'A <br /> iff code :: [ 'a -> 'b ]
|o| LE                      | int : 'S -> bool : 'S
|o| LEFT 'b                 | 'a : 'S -> or 'a 'b : 'S
|o| LEVEL                   | 'S -> nat : 'S
|o| LOOP body               | bool : 'A -> 'A <br /> iff   body :: [ 'A -> bool : 'A ]
|o| LOOP_LEFT body          | (or 'a 'b) : 'A -> 'b : 'A <br /> iff   body :: [ 'a : 'A -> (or 'a 'b) : 'A ]
| | LSL                     | nat : nat : 'S -> nat : 'S
| | LSR                     | nat : nat : 'S -> nat : 'S
|o| LT                      | int : 'S -> bool : 'S
| | MAP body                | (map 'key 'val) : 'A -> (map 'key 'b) : 'A <br /> iff   body :: [ (pair 'key 'val) : 'A -> 'b : 'A ]
| |                         | (list 'elt) : 'A -> (list 'b) : 'A <br /> iff   body :: [ 'elt : 'A -> 'b : 'A ]
|o| MEM                     | 'elt : set 'elt : 'S -> bool : 'S
|o|                         | 'key : map 'key 'val : 'S -> bool : 'S
|o|                         | 'key : big_map 'key 'val : 'S -> bool : 'S
|o| MUL                     | int : int : 'S -> int : 'S
|o|                         | int : nat : 'S -> int : 'S
|o|                         | nat : int : 'S -> int : 'S
|o|                         | nat : nat : 'S -> nat : 'S
|o|                         | mutez : nat : 'S -> mutez : 'S
|o|                         | nat : mutez : 'S -> mutez : 'S
| |                         | bls12_381_g1 : bls12_381_fr : 'S -> bls12_381_g1 : 'S
| |                         | bls12_381_g2 : bls12_381_fr : 'S -> bls12_381_g2 : 'S
| |                         | bls12_381_fr : bls12_381_fr : 'S -> bls12_381_fr : 'S
| |                         | nat : bls12_381_fr : 'S -> bls12_381_fr : 'S
| |                         | int : bls12_381_fr : 'S -> bls12_381_fr : 'S
| |                         | bls12_381_fr : nat : 'S -> bls12_381_fr : 'S
| |                         | bls12_381_fr : int : 'S -> bls12_381_fr : 'S
|o| NEG                     | int : 'S -> int : 'S
|o|                         | nat : 'S -> int : 'S
| |                         | bls12_381_g1 : 'S -> bls12_381_g1 : 'S
| |                         | bls12_381_g2 : 'S -> bls12_381_g2 : 'S
| |                         | bls12_381_fr : 'S -> bls12_381_fr : 'S
|o| NEQ                     | int : 'S -> bool : 'S
| | NEVER                   | never : 'A -> 'B
|o| NIL 'a                  | 'S -> list 'a : 'S
|o| NONE 'a                 | 'S -> option 'a : 'S
|o| NOT                     | bool : 'S -> bool : 'S
| |                         | nat : 'S -> int : 'S
| |                         | int : 'S -> int : 'S
|o| NOW                     | 'S -> timestamp : 'S
|o| OR                      | bool : bool : 'S -> bool : 'S
| |                         | nat : nat : 'S -> nat : 'S
|o| PACK                    | 'a : 'S -> bytes : 'S
|o| PAIR                    | 'a : 'b : 'S -> pair 'a 'b : 'S
|o| PAIR n                  | 'a{0} : ... : 'a{n-1} : 'A -> pair 'a{0} ...  'a{n-1} : 'A
| | PAIRING_CHECK           | list (pair bls12_381_g1 bls12_381_g2) : 'S -> bool : 'S
|o[^push]| PUSH 'a x               | 'A -> 'a : 'A <br /> iff   x :: 'a
| | READ_TICKET             | ticket 'a : 'S -> pair address 'a nat : ticket 'a : 'S
|o| RENAME                  | 'S -> 'S
|o| RIGHT 'a                | 'b : 'S -> or 'a 'b : 'S
| | SAPLING_EMPTY_STATE ms  | 'S -> sapling_state ms: 'S
| | SAPLING_VERIFY_UPDATE   | sapling_transaction ms : sapling_state ms : 'S -> option (pair int (sapling_state ms)): 'S
|o| SELF                    | 'S -> contract 'p : 'S
|o| SELF_ADDRESS            | 'S -> address : 'S
|o| SENDER                  | 'S -> address : 'S
|o| SET_DELEGATE            | option key_hash : 'S -> operation : 'S
|o| SHA256                  | bytes : 'S -> bytes : 'S
|o| SHA3                    | bytes : 'S -> bytes : 'S
|o| SHA512                  | bytes : 'S -> bytes : 'S
|o| SIZE                    | string : 'S -> nat : 'S
|o|                         | set 'elt : 'S -> nat : 'S
|o|                         | map 'key 'val : 'S -> nat : 'S
|o|                         | list 'elt : 'S -> nat : 'S
|o|                         | bytes : 'S -> nat : 'S
|o| SLICE                   | nat : nat : string : 'S -> option string : 'S
|o|                         | nat : nat : bytes : 'S -> option bytes : 'S
|o| SOME                    | 'a : 'S -> option 'a : 'S
|o| SOURCE                  | 'S -> address : 'S
| | SPLIT_TICKET            | ticket 'a : (pair nat nat) : 'S -> option (pair (ticket 'a) (ticket 'a)) : 'S
|o| SUB                     | int : int : 'S -> int : 'S
|o|                         | int : nat : 'S -> int : 'S
|o|                         | nat : int : 'S -> int : 'S
|o|                         | nat : nat : 'S -> int : 'S
|o|                         | timestamp : int : 'S -> timestamp : 'S
|o|                         | timestamp : timestamp : 'S -> int : 'S
|o|                         | mutez : mutez : 'S -> mutez : 'S
|o| SWAP                    | 'a : 'b : 'A -> 'b : 'a : 'A
| | TICKET                  | 'a : nat : 'S -> ticket 'a : 'S
|o| TOTAL_VOTING_POWER      | 'S -> nat : 'S
|o| TRANSFER_TOKENS         | 'p : mutez : contract 'p : 'S -> operation : 'S
|o| UNIT                    | 'A -> unit : 'A
|o| UNPACK 'a               | bytes : 'S -> option 'a : 'S
|o| UNPAIR                  | pair 'a 'b : 'S -> 'a : 'b : 'S
|o| UNPAIR n                | pair 'a{0} ... 'a{n-1} : S -> 'a{0} : ... : 'a{n-1} : S
|o| UPDATE                  | 'elt : bool : set 'elt : 'S -> set 'elt : 'S
|o|                         | 'key : option 'val : map 'key 'val : 'S -> map 'key 'val : 'S
|o|                         | 'key : option 'val : big_map 'key 'val : 'S -> big_map 'key 'val : 'S
|o| UPDATE 0                | 'a : 'b : 'S -> 'a : 'S
|o| UPDATE (2k)             | 'c : pair 'a{0} ... 'a{k-1} 'a{k} : 'S -> pair 'a{0} ... 'a{k-1} 'c : 'S
|o| UPDATE (2k+1)           | 'c : pair 'a{0} ... 'a{k} 'a{k+1} : 'S -> pair 'a{0} ... 'a{k-1} 'c 'a{k+1} : 'S
|o| VOTING_POWER            | key_hash : 'S -> nat : 'S
|o| XOR                     | bool : bool : 'S -> bool : 'S
| |                         | nat : nat : 'S -> nat : 'S

[^compare]: Only equality is concerned.
[^create_contract]: Supported but not well worked because of tezos-client side's bug.
[^push]: `set`, `map`, and `lambda` are not supported.

Deprecated
- CREATE_ACCOUNT
- STEPS_TO_QUOTA
