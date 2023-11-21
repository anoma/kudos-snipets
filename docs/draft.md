
```juvix
module Kudo;
  
  type Resource :=
    mkResource {
      label : Hash;
      predicate : Predicate;
      quantity : Field;
      value : Value
    };

  type Value :=
    mkValue {
      owner : Owner;
      originator : Signature
    };

--   Env : Type := Set Predicate;
end;
```


```juvix
trait
type isVerifiable (A : Type) :=
  mkVerifiable {verify : A -> Bool};

open isVerifiable;

-- Value can be verified, say, trivially for the moment.
instance
valueIsVerifiable : isVerifiable Kudo.Value :=
  mkVerifiable (verify := \ {_ := true});
```

```juvix
kudoIsVerifiable : isVerifiable Kudo.Resource :=
  mkVerifiable
    (verify := \ {k := verify (Kudo.Resource.value k)});
```

### Partial transactions


```juvix
type PartialTransaction :=
  mkPartialTransaction {
    input : List Kudo.Resource;
    output : List Kudo.Resource;
    extra : Bytes
  };
open PartialTransaction;
```

## Evaluation context

The evaluation context (ctx) consists of a single PTX and some "system calls" providing extra functionality 
we do not want to, or can not implement inside of a resource logic.
During evaluation, the predicates from all resources get applied to the PTX one after another.

"Syscalls":

- self_hash: since some logics will need to know about their own hash/address, and trying to compute the hash from 
inside the logic ("self.hash(self)") is not possible since there is no fixed point, the ctx needs to provide the 
hash of each resource_logic to itself. 
The ctx would then replace the term "self_hash" in a resource_logic by the corresponding hash during evaluation.

- verify: since we do not want to implement cryptographic primitives in the logics, the ctx should provide them by 
taking (Data, Key) as arguments and returning Bool. For the prototype, keys can be of type Bool with 
check_signature just returning the value from the key.

```juvix
{-
axiom env : Kudo.Env;
axiom ! : {A : Type} -> A;

-- The following is called at validation time (taiga to be used in the shielded case).
kudo_logic (ptx : PartialTransaction) : Bool :=
  let
    checkInputs : Bool :=
      all
        λ {ri :=
          let
            cond1 : Bool := member? (Kudo.Resource.predicate ri) env;
            cond2 : _ := true;
            cond3 : _ := true;
          in cond1 && not (cond2 || cond3)}
        (input ptx);
    checkOutput : Bool :=
      all
        λ {ri :=
          let
            cond1 : Bool := member? (Kudo.Resource.predicate ri) env;
            cond2 : _ := true;
            cond3 : _ := true;
          in cond1 && not (cond2 || cond3)}
        (output ptx);
    checkConnectiveValidity : Bool := true;
  in checkInputs && checkOutput && checkConnectiveValidity;

-- insert kudo_logic in the set, 
-- or better use a map instead

instance
partialVerifiable : isVerifiable PartialTransaction :=
  mkVerifiable (verify := kudo_logic);
-}
```

### Example Resources


### Intents over Kudos

 To encode simple preferences for independent Resources, agents can weigh each Resource. 
 This encodes how much they value having a certain Resource. 
 In other words: For Resources they already have Agents prefer to not trade ones with high weights.
      Inversely, for Resources they want they prefer to trade for the ones with high weights.

 This enables, e.g. a CSP solver to optimize outcomes locally in respect to the assignment of the formula
 by scaling boolean variables.

A general Intent would look like this:


- *Note*: Want ResourceType is only used for output resource types, to get the owner right, balance checks enforce inputs of the same type to exist.


- *Note*: We might also introduce HaveResourceType later, in case agents want to publish intents e.g. for 3rd parties to perform a swap.

