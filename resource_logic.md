# Resource Logic

What follows is an explanation of the encoding of Kudos and Intents over them as `Resources` with a corresponding `Resource Logic`. 

```rust
// Kudos
// Multi-Agent, Multi-Resource swaps over Resource Types tied to Identities

// -- Kudos as Resources --

struct KudoResource {
    predicate: Predicate,   // kudo_logic
    label: FFElement,       // (Hash of the) ExternalIdentity of the Originator, might need to be resolved to the actual ExtID
    quantity: FFElement,    // Encoding of Nat? Or are they also be native FFElements?
    value: KudoValue,       // Application Data
} // Predicate and Value will be encoded as Program Field Elements

// Kudos are fungible iff (kudoA1.predicate == kudoA2.predicate && kudoA1.label == kudoA2.label)

struct KudoValue {
    owner: ExternalIdentity,        
    owner_sig: Signature,
    originator_sig: Signature, // Signature over Predicate and Label (Resource Type)
}

struct ExternalIdentity {
    key: Key,
}

impl ExternalIdentity {
    fn verify(self, bytes: [Byte], sig: Signature) -> bool {
        // if sig is valid signature from self
        //    return true
        // otherwise
        //    return false
    }
}

struct PTX {
    input_resources: Vec<Resource>,
    out_resources: Vec<Resource>,
    extra_data: Vec<Byte>,
}

// Evaluation context
// The evaluation context (ctx) consists of a single PTX and some "system calls" providing extra functionality 
// we do not want to, or can not implement inside of a resource logic.
// During evaluation, the predicates from all resources get applied to the PTX one after another.
//
// "Syscalls":
// - self_hash: since some logics will need to know about their own hash/address, and trying to compute the hash from 
//   inside the logic ("self.hash(self)") is not possible since there is no fixed point, the ctx needs to provide the 
//   hash of each resource_logic to itself. 
//   The ctx would then replace the term "self_hash" in a resource_logic by the corresponding hash during evaluation.
// - verify: since we do not want to implement cryptographic primitives in the logics, the ctx should provide them by 
//   taking (Data, Key) as arguments and returning Bool. For the prototype, keys can be of type Bool with 
//   check_signature just returning the value from the key.



// The following is called at validation time (taiga to be used in the shielded case).
fn kudo_logic(ptx: Ptx) -> bool {
    for ri in ptx.resources.input {
        if ri.predicate == env.hash_self { // if the Resource is a Kudo
            if not ri.label.verify([ri.predicate.as_bytes()++ri.label.as_bytes()], r.value.originator_sig) {
                return false // fail on invalid originator 1/2
            } // check if Value contains a valid signature of the originator, denoted by the label
              // since label is external Identity, we can call verification of a signature
        if not ri.value.owner.verify(ri.as_bytes, ri.value.owner_sig) {  
                return false  // if sig is valid, owner has authorized consumption
            }
        }
        // Check:
        // If a kudo without owner_sig is consumed, the only kudo that can be created is one that has the same contents + owner_sig
    }

    for ro in ptx.resources.output {
        if ro.predicate == env.hash_self {
            if ro.label.verify(r.value.originator_sig) == false {
                return false // fail on invalid Originator 2/2
            }
        }
        // Check:
        // Balance per owner and resource type should be implicit to authorized consumption + balance check per type 
    }

    // Check connective validity
    if ptx.extra_data.connectives == false {
        return false
    }
    
    return true
} 
// In addition to validating the resource logic, balance checks per resource type are always performed. 

// -- Example Resources --
let valueA1 = KudoValue {
    owner: agentA,
    owner_sig: sig_agentA1,
    originator_sig: type_sig_agentA1,
}

let kudoA1 = KudoResource {
    predicate: kudo_logic,
    label: hash(agentA)
    quantity: 1,
    value: valueA1,
}

let valueA2 = KudoValue {
    owner: agentA,
    owner_sig: sig_agentA2,
    originator_sig: type_sig_agentA2,
}

let kudoA2 = KudoResource {
    predicate: kudo_logic,
    label: hash(agentA)
    quantity: 1,
    value: valueA2,
}

let valueB1 = KudoValue {
    owner: agentB,
    owner_sig: sig_agentB1,
    originator_sig: type_sig_agentB1,
}

let kudoB1 = KudoResource {
    predicate: kudo_logic,
    label: hash(agentB)
    quantity: 1,
    value: valueB1,
}
// Analogously for B2, C1, C2

// -- Intents over Kudos --

// To encode simple preferences for independent Resources, agents can weigh each Resource. 
// This encodes how much they value having a certain Resource. 
// In other words: For Resources they already have Agents prefer to not trade ones with high weights.
//      Inversely, for Resources they want they prefer to trade for the ones with high weights.
//
// This enables, e.g. a CSP solver to optimize outcomes locally in respect to the assignment of the formula
// by scaling boolean variables.

type Weight = Float;

// A general Intent would look like this:
enum ResourceIntentElement { // TODO: Find a better name, if we think this is an important object 
    Resource,         // This could be a resource that an agent owns, or a fully specified resource they want.
    WantResourceType  // This specifies the Type (Predicate + Label) of a Resource an agent wants and requires Value.owner set to the intent creating agent.
}
// Note: Want ResourceType is only used for output resource types, to get the owner right, balance checks enforce inputs of the same type to exist.

// Note: We might also introduce HaveResourceType later, in case agents want to publish intents e.g. for 3rd parties to perform a swap.

// Agents might want to express inclusionary (OR) or exclusionary (XOR) preferences. 
// Inclusionary means they want to give or get either or all resources from a clause.
// Exclusionary means they want to give or get exactly one subclause.
// To be compatible with the CSP backend these should have a standard form as follows:
//
// let have0..2, want0..2 be ResourceIntentElements
//
// have: (have0) XOR (have1 OR have2)
// want: (want0 OR want1) XOR (want2)
//
// During validation of the connective, only the Resource(Type) and have/want property should be relevant, 
// i.e. validation should be invariant against the relative positions of haveI, haveJ. 
//
// TODO: We should specify this more precisely, e.g. via a BNF
type Connective = Term;

// Intents like the following (or a flat string representation as above) would be generated to be ingested by solvers, 
// which in turn produce transactions to be validated. Validation and solving time should be fully disjoint in practice. 
struct ResourceIntent {
    have: Vec<(Weight, ResourceIntentElement)>,
    want: Vec<(Weight, ResourceIntentElement)>,
    connectives: Connective,
}
// This would be encoded as a PTX as follows:
// - elements of have as input_resources
// - elements of want as ephemeral output_resource 
// - ephemeral resources required for balancing on both sides are inferred
// - solver hints encode the weights and connectives in ptx.extra_data
//
// Note: The layout here is kept close to what a PTX looks like for clarity of explanation,
// but more concise representations can be chosen for Juvix in the service of improving UX.

// Since Agents might know which type of Kudo they seek, but not all the details of the Resource, 
// we want to enable intents over Kudo Resource Types.
enum KudoIntentElement {
    KudoResource,
    WantKudoResourceType, // As WantResourceType, but with Predicate = kudo_logic
}

// Each Agent can produce intents over which Kudos they have and which Kudos they want, for arbitrary size vectors of both.
struct KudoIntent {
    have: Vec<(Weight, KudoIntentElement)>,
    want: Vec<(Weight, KudoIntentElement)>
    connectives: Connective,
} 
// For an example intent and solver see: https://github.com/anoma/kudos-snippets/blob/main/intents_stand_mix.ipynb

// Functionality of the VM that is needed:
// - signature scheme accessible via opcode
// - predicate.hash(self) provided via "syscall"
// - API to "send" a kudo to someone
// - API to check kudos I have

// TODOs for V2:
// - multi round solving/pathfinding, in case intents can not be satisfied locally on a single solver
// - bundle splitting, to handle resources with quantity > 1 
```