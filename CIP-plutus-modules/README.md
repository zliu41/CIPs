---
CIP: "?"
Title: Modules in UPLC
Status: Proposed
Category: Plutus
Authors:
  - John Hughes <john.hughes@quviq.com>
Implementors: []
Discussions: []
Created: 2024-11-12
License: CC-BY-4.0
---
## Abstract
TODO: write an abstract (~200 words) summarizing the CIP.

## Motivation: why is this CIP necessary?

Cardano scripts are currently subject to a fairly tight size limit;
even when they are supplied as a reference input, that UTxO must be
created by a single transaction, which is subject to the overall
transaction size limit. Competing blockchains suffer from no such
limit: on the Ethereum chain, for example, contracts can call one
another, and so the code executed in one transaction may come from
many different contracts, created independently on the
blockchain--each subject to a contract size limit, but together
potentially many times that size. This enables more sophisticated
contracts to be implemented; conversely, on Cardano, it is rather
impractical to implement higher-level abstractions as libraries,
because doing so will likely exceed the script size limit. This is not
just a theoretical problem: complaints about the script size limit are
the most common complaint made by Cardano contract developers.

Thus the primary goal of this CIP is to lift the limit on the total
amount of code run during a script execution, by allowing part of the
code to be provided in external modules. By storing these modules on
the blockchain and providing them as reference UTxOs, it will be
possible to keep transactions small even though they may invoke a large
volume of code.

Once scripts can be split into separate modules, then the question
immediately arises of whether or not the script and the modules it
imports need to be in the same language. Today there are many
languages that compile to UPLC, and run on the Cardano
blockchain. Ideally it should be possible to define a useful library
in any of these languages, and then use it from all of them. A
secondary goal is thus to define a module system which permits this,
by supporting cross-language calls.

Note that Plinth already enjoys a module system, namely the Haskell
module system. This already enables Plinth code to be distributed
across several modules, or put into libraries and shared. Indeed
this is already heavily used: the DJED code base distributes Plinth
code across 24 files, of which only 4 contain top-level contracts, and
the others provide supporting code of one sort or another. Thus the
software engineering benefits of a module system are already
available; other languages compiled to UPLC could provide a module
system in a similar way. The *disadvantage* of this approach is that
all the Plinth code is put together into one script, which can easily
exceed the size limit. Indeed, the DJED code base also contains an
implementation of insertion sort in Plinth, with a comment that a
quadratic algorithm is used because its code is smaller than, for
example, QuickSort. There is no clearer way to indicate why the
overall size limit must be lifted.

### The Situation on Ethereum

Ethereum contracts are not directly comparable to Cardano scripts;
they correspond to both the *on-chain* and the *off-chain* parts of
Cardano contracts, so one should expect Ethereum contracts to require
more code for the same task, since in Cardano only the verifiers need
to run on the chain itself. Nevertheless, it is interesting to ask
whether, and how the need for modules has been met in the Ethereum
context.

Solidity does provide a notion of 'library', which collects a number
of reusable functions together. Libraries can be 'internal' or
'external'--the former are just compiled into the code of client
contracts (and so count towards its size limit), while the latter are
stored separately on the blockchain.

There is a problem of trust in using code supplied by someone else:
the documentation for the `ethpm` package manager for Ethereum warns
sternly

**you should NEVER import a package from a registry with an unknown or
  untrusted owner**

It seems there *is* only one trusted registry, and it is the one
supplied as an example by the developers of `ethpm`. In other words,
while there is a package manager for Ethereum, it does not appear to
be in use.

This is not to say that code is never shared. On the contrary, there
is an open source repo on `github` called `OpenZeppelin` which appears
to be heavily used. It provides 264 Solidity files, in which 43
libraries are declared (almost all internal). It seems, thus, that
libraries are not the main way of reusing code in Solidity; rather it
is by calling, or inheriting from, another contract, that code reuse
primarily occurs.

A small study of 20 'verified' contracts running on the Ethereum chain
(verified in the sense that their source code was provided) showed that

* 55% of contracts consisted of more than one module
* 40% of contracts contained more than one 'application' module
* 55% of contracts imported `OpenZeppelin` modules
* 10-15% of contracts imported modules from other sources
* 5% of contracts were simply copies of `OpenZeppelin` contracts

Some of the 'other' modules were provided to support specific
protocols; for example Layr Labs provide modules to support their
Eigenlayer protocol for re-staking.

A sample of 20 is too small to draw very strong statistical conclusions,
but we can say that the 95% confidence interval for contracts to
consist of multiple modules is 34-74%.
Thus code sharing is clearly going on, and a significant number of
transactions exploit multiple modules. We may conclude that there is a
significant demand for modules in the context of smart contracts, even
if the total contract code still remains relatively small.


## Specification

### Adding modules to UPLC

This CIP provides the simplest possible way to split scripts across
multiple UTxOs; essentially, it allows any closed subterm to be
replaced by its hash, whereupon the term can be supplied either as a
witness in the invoking transaction, or via a reference script in that
transaction. To avoid any change to the syntax of UPLC, hashes are
allowed only at the top-level (so to replace a deeply nested subterm
by its hash, we need to first lambda-abstract it). This also places
all references to external terms in one place, where they can easily
be found and resolved. Thus we need only change the definition of a
`Script`; instead of simply some code, it becomes the application of
code to zero or more arguments, given by hashes.

Currently, the definition of “script” used by the ledger is (approximately):
```
newtype Script = Script ShortByteString
```
We change this to:
```
newtype CompleteScript = CompleteScript ShortByteString

newtype Arg = ScriptArg ScriptHash

data Script = 
  ScriptWithArgs { head :: CompleteScript, args :: [Arg] }

-- hash of a Script, not a CompleteScript
type ScriptHash = ByteString
```
We need to resolve the arguments of a script before running it:
```
resolveScriptDependencies 
  :: Map ScriptHash Script
  -> Script 
  -> Maybe CompleteScript
resolveScriptDependencies preimages = go 
  where 
    go (ScriptWithArgs head args) = do
      argScripts <- traverse lookupArg args
      pure $ applyScript head argScripts
      where
        lookupArg :: Arg -> Maybe CompleteScript 
        lookupArg (ScriptArg hash) = do
          script <- lookup hash preimages
          go script
```
The `preimages` map is the usual witness map constructed by the ledger,
so in order for a script hash argument to be resolved, the transaction
must provide the pre-image in the usual way. Note that arguments are
mapped to a `Script`, not a `CompleteScript`, so the result of looking
up a hash may contain further dependencies, which need to be resolved
recursively. A transaction must provide witnesses for *all* the
recursive dependencies of the scripts it invokes.

The only scripts that can be run are complete scripts, so the type of
`runScript` changes to take a `CompleteScript` instead of a `Script`.

#### Variation

With this design, if any script hash is missing from the `preimages`,
then the entire resolution fails. As an alternative, we might replace
missing subterms by a dummy value, such as `builtin unit`, thus:
```
resolveScriptDependencies 
  :: Map ScriptHash Script
  -> Script 
  -> CompleteScript
resolveScriptDependencies preimages = go 
  where 
    go (ScriptWithArgs head args) =
      applyScript head (map lookupArg args)
      where
        lookupArg :: Arg -> CompleteScript 
        lookupArg (ScriptArg hash) = do
          case lookup hash preimages of
	    Nothing     -> builtin unit
	    Just script -> go script
```
This would allow transactions to provide witnesses only for script
arguments which are actually *used* in the calls that the transaction
makes. This may sometimes lead to a significant reduction in the
amount of code that must be loaded; for example, imagine a spending
verifier which offers a choice of two encryption methods, provided as
separate script arguments. In any call of the verifier, only one
encryption method will be required, allowing the other (and all its
dependencies) to be omitted from the spending transaction.


### Modules in TPLC

No change is needed in TPLC.

### Modules in PIR

No change is needed in PIR.

### Modules in Plinth

The Plinth modules introduced in this CIP bear no relation to Haskell
modules; their purpose is simply to support the module mechanism added
to UPLC. They are first-class values in Haskell.

Just as we introduced a distinction in UPLC between `CompleteScript`
and `Script`, so we introduce a distinction in Plinth between
`CompiledCode a` (returned by the Plinth compiler when compiling a
term of type `a`), and `Module a` representing a top-level `Script`
with a value of type `a`.
```
newtype Module a = Module {unModule :: Mod}

data Mod = forall b. Mod{ modCode :: CompiledCode b,
     	   	     	  modArgs :: [Mod],
			  modHash :: ScriptHash }
```

Here the `modArgs` correspond to the `ScriptArg`s in the UPLC case,
and the `modHash` is the hash of the underlying `Script`.  The type
parameter of `Module a` is a phantom parameter, just like the type
parameter of `CompiledCode a`, which tells us the type of value which
the application of the `modCode` to the `modArgs` represents. Notice
that the module arguments will usually be of different types, and so
they are stored in a list as the underlying representation type `Mod`.

We need a way to convert `CompiledCode` into a `Module`:
```
makeModule :: CompiledCode a -> Module a
makeModule code = Module (Mod code [] ...compute the script hash...)
```

and we also need a way to supply an imported module to a `Module`:
```
applyModule :: Module (a->b) -> Module a -> Module b
applyModule (Module (Mod code args _)) m =
  Module (Mod code (args++[m]) ...compute the script hash...)
```

As in UPLC, the intention is that scripts that import modules be
written as lambda-expressions, and the imported module is then
supplied using `applyModule`. No change is needed in the Plinth
compiler to support this mechanism.

It is `Module` values that would then be serialised to produce scripts
for inclusion in transactions.

## Rationale: how does this CIP achieve its goals?

This CIP provides a minimal mechanism to split scripts across several
transactions. 'Imported' modules are provided in the calling
transaction and passed as arguments to the top-level script, and their
identity is checked using their hash. The representation of modules is
left entirely up to compiler-writers to choose--a module may be any
value at all. For example, one compiler might choose to represent
modules as a tuple of functions, while another might map function
names to tags, as Solidity does, and represent a module as a function
from tags to functions. Each language will need to define its own
conventions for module representations, and implement them on top of
this low-level mechanism. For example, a typed language might
represent a module as a tuple of exported values, and store the names
and types of the values in an (off-chain) interface file. Clients
could use the interface file to refer to exported values by name, and
to perform type-checking across module boundaries.

### Recursive modules

This design does not support mutually recursive modules. Module
recursion is sometimes used in languages such as Haskell, but it is a
rarely-used feature that will not be much missed.

### Cross-language calls

There is no a priori reason why script arguments need be written in
the same high-level language as the script itself; thus this CIP
supports cross-language calls. However, since different languages may
adopt different conventions for how modules are represented, then some
'glue code' is likely to be needed for modules in different languages
to work together. In the longer term, it might be worthwhile defining
an IDL (Interface Definition Language) for UPLC, to generate this glue
code, and enable scripts to call code in other languages more
seamlessly. This is beyond the scope of this CIP; however this basic
mechanism will not constrain the design of such an IDL in the future.

In Plinth, because the `Module` type is a phantom type, it is easy to
take code from elsewhere and turn it into a `Module t` for arbitrary
choice of `t`; this can be used to import modules compiled from other
languages into Plinth (provided a sensible Plinth type can be given to
them).


### Static vs Dynamic Linking

With the introduction of modules, scripts are no longer
self-contained--they may depend on imported modules. This applies both
to scripts for direct use, such as spending verifiers, and to scripts
representing modules stored on the chain.  A module may depend on
imported modules, and so on transitively. An important question is
when the identity of those modules is decided. In particular, if a
module is replaced by a new version, perhaps fixing a bug, can
*existing* code on the chain use the new version instead of the old?

The design in this CIP supports both. Suppose a module `A` imports
modules `B` and `C`. Then module `A` will be represented as the
lambda-expression `λB.λC.A`. This can be compiled into a
`CompleteScript` and placed on the chain, with no `ScriptArg`s, as a
reference script in a UTxO, allowing it to be used with any
implementations of `B` and `C`--the implementations must just be
provided in the calling script. We call this 'dynamic linking',
because the implementation of dependencies may vary from use to
use. On the other hand, if we want to *fix* the versions of `B` and
`C` then we can create a `Script` that applies the same
`CompleteScript` to two `ScriptArg`s, containing the hashes of the
intended versions of `B` and `C`, which will then be supplied by
´resolveScriptDependencies`. We call this 'static linking', because
the version choice for the dependency is fixed by the scripe. It is up
to compiler writers to decide between static and dynamic linking
in this sense.

On the other hand, when a script is used directly as a verifier then
there is no opportunity to supply additional arguments; all modules
used must be supplied as `ScriptArg`s, which means they are
fixed. This makes sense: it would be perverse if a transaction trying
to spend a UTxO protected by a verifier were allowed to replace some
of the code in the verifier--that would open a can of worms,
permitting many attacks whenever a script was split over several
modules. With the design in the CIP, it is the script in the UTxO that
determines the module versions to be used, not the spending
transaction. That transaction does need to supply all the modules
actually used--including all the dependencies--but cannot choose to
supply alternative implementations of them.

### In-service upgrade

Long-lived contracts may need upgrades and bug fixes during their
lifetimes. This need is met on the Ethereum blockchain using the
'proxy pattern'--a 'proxy' contract which delegates calls to the
current implementation contract, whose identity is stored in the proxy
contract's mutable state. Proxy contracts can provide a 'code upgrade'
method which stores a new implementation contract in the mutable
state.

The Cardano chain does not offer mutable state. Instead, a changing
state is represented by a succession of UTxOs, each holding the
current state, usually with the currently-valid UTxO identified by
holding a particular NFT. In the absence of mutable state a dependency
cannot be updated just be changing a pointer, but scripts can still
be upgraded by creating new values on the chain. The exact mechanism
depends on the kind of script--and, often, on the original
script developer preparing the ground for a later code change.

First consider modules, stored as reference scripts in UTxOs. The hash
of a module depends on the hash of all its dependencies, so when a
dependency changes, then a new version of the UTxO needs to be created
with the new dependency, and its hash needs to be distributed (by
off-chain means). To prevent accidental use of the old UTxO, it could
be spent.

UTxOs whose *spending verifier* needs upgrading can be spent and
recreated with a new verifier, if the need has been anticipated by the
script author. The verifier would need to accept a 'code change'
redeemer, and then check that the transaction created a new UTxO
protected by the new spending verifier. For example, the code change
redeemer might provide the hash of the new verifier, and the old
verifier would then check that the new UTxO was protected by that
hash. This mechanism permits an arbitrary code change; of course this
opens for attacks, so in practice such a verifier would need to check
that the proposed code change was correctly authorised. How this is
done is up to the contract concerned.

Note that *currency symbols* in Cardano are just the hash of the
minting policy--a script. Thus, updating a dependency of a minting
policy means changing the currency symbol. We need to be able to
convert tokens with the old currency to the new one. To allow this,
the minting script must allow *burning* the old currency when the new
one is being minted, and the new minting script must allow minting
when the old currency is being burned, provided the code upgrade is
correctly authorized. If the currency is to be stored in UTxOs
protected by spending verifiers, then those verifiers must also accept
'currency upgrade' redeemers, and check that the UTxO is just being
recreated with old tokens replaced by new ones.

Staking validators are a simple case: they can be upgraded just by
deregistering the state key registration certificate that refers to
them, and then reregistering the same state key with a new staking
validator.

### Lazy loading

The variation in the specification section above permits what we call
'lazy loading'.  Dependency trees have a tendency to grow very large;
when one function in a module uses another module, it becomes a
dependency of the entire module and not just of that function. It is
easy to imagine situations in which a script depends on many modules,
but a particular call requires only a few of them. For example, if a
script offers a choice of protocols for redemption, only one of which
is used in a particular call, then many modules may not actually be
needed. The variation allows a transaction to omit the unused modules
in such cases. This reduces the size of the transaction, which need
provide fewer witnesses, but more importantly it reduces the amount of
code which must be loaded from reference UTxOs.

If a script execution *does* try to use a module which was not
provided, it will encounter a run-time type error and fail (unless the
module value was `builtin unit`, in which case the script will behave
as though the module had been provided).

To take advantage of this variation, it is necessary, when a
transaction is constructed, to *observe* which script arguments are
actually used by the script invocations needed to validate the
transaction. The transaction balancer runs the scripts anyway, and so
can in principle observe the uses of script arguments, and include
witnesses in the transaction for just those arguments that are used.
Suitable balancer modifications to achieve this are not part of this
CIP.


### Transaction fees

Imported modules are provided using reference scripts, an existing
mechanism (see CIP-33), or in the transaction itself. Provided the
cost of loading reference scripts is correctly accounted for, this CIP
introduces no new problems.

Note that there is now (since August 2024) a hard limit on the total
size of reference scripts used in a transaction, and the transaction
fee is exponential in the total script size (see
[here](https://github.com/IntersectMBO/cardano-ledger/blob/master/docs/adr/2024-08-14_009-refscripts-fee-change.md)).The
exponential fees provide a strong motivation to prefer the 'lazy
loading' variation in this CIP: even a small reduction in the number
of reference scripts that need to be provided may lead to a large
reduction in transaction fees.

The motivation for these fees is to deter DDoS attacks based on
supplying very large Plutus scripts that are costly to deserialize,
but run fast and so incur low execution unit fees. While these fees
are likely to be reasonable for moderate use of the module system, in
the longer term they could become prohibitive for more complex
applications. It may be necessary to revisit this design decision in
the future. To be successful, the DDoS defence just needs fees to
become *sufficiently* expensive per byte as the total size of
reference scripts grows; they do not need to grow without bound. So
there is scope for rethinking here.

### Verification

Since scripts invoked by a transaction specify all their dependencies
as hashes, then the running code is completely known, and testing or
formal verification is no harder than usual. Standalone verification
of modules using 'dynamic linking' poses a problem, however, in that
the code of the dependencies is unknown. This makes testing
difficult--one would have to test with mock implementations of the
dependencies--and formal verification would require formulating
assumptions about the dependencies that the module can rely on, and
later checking that the actual implementations used fulfill those
assumptions.

### Impact on optimisation and scipt performance

Splitting a script into separately compiled parts risks losing
optimisation opportunities that whole-program compilation gives. Note
that script arguments are known in advance, and so potentially some
cross-module optimisation may be possible, but imported modules are
shared subterms between many scripts, and they cannot be modified when
the client script is compiled. Moreover, unrestrained inlining across
module boundaries could result in larger script sizes, and defeat the
purpose of breaking the code into modules in the first place.

On the other hand, since the size limit on scripts will be less of a
problem, then compilers may be able to optimize *more*
aggressively. For example, today the Plinth inliner is very careful
not to increase script size, but once modules are available it may be
able to inline more often, which can enable further optimizations.

Moreover, today we see examples of deliberate choice of worse
algorithms, because their code is smaller. Removing the need to make
such choices can potentially improve performance considerably.


## Path to Active
### Acceptance criteria
TODO: The criteria whereby the proposal becomes 'Active'.
### Implementation Plan
TODO: A plan to meet the criteria or N/A
## Categories
?
I'm unsure whether this is a Plutus CIP or a Ledger CIP.
## Versioning
## References
## Appendices
## Acknowledgements

This CIP draws heavily on a design by Michael Peyton Jones.

## Copyright
This CIP is licensed under [CC-BY-4.0]](https://creativecommons.org/licenses/by/4.0/legalcode).
