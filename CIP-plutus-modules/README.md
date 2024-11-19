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

Thus code sharing is clearly going on, and a (small) majority of
transactions exploit multiple modules. We may conclude that there is a
significant demand for modules in the context of smart contracts, even
if the total contract code still remains relatively small.


## Specification

This CIP provides the simplest possible way to split scripts across
multiple UTxOs; essentially, it allows any closed subterm to be
replaced by its hash, whereupon the term can be supplied either as a
witness in the invoking transaction, or via a reference script in that
transaction. To avoid any change to the syntax of UPLC, hashes are
allowed only at the top-level (so to replace a deeply nested subterm
by its hash, we need to first lambda-abstract it). Thus we need only
change the definition of a `Script`; instead of simply some code, it
becomes the application of code to zero or more arguments, given by
hashes.

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

### Variation

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

If the need arises to upgrade a module used in a spending verifier,
then that must be achieved by spending the UTxO concerned, and
creating a new one that depends on the new module version. That is,
the potential for code upgrade must be built into the spending
verifier, allowing checks that the upgrade is authorised to be
performed by the contract.

### Lazy loading

Dependency trees have a tendency to grow very large; when one function
in a module uses another module, it becomes a dependency of the entire
module and not just of that function. It is easy to imagine situations
in which a script depends on many modules, but a particular call
requires only a few of them. For example, if a script offers a choice
of protocols for redemption, only one of which is used in a particular
call, then many modules may not actually be needed. The variation
above allows a transaction to omit the unused modules in such
cases. This reduces the size of the transaction, which need provide
fewer witnesses, but more importantly it reduces the amount of code
which must be loaded from reference UTxOs.

To take advantage of this possibility, it is necessary, when a
transaction is constructed, to *observe* which script arguments are
actually used by the script invocations needed to validate the
transaction. The transaction balancer runs the scripts anyway, and so
can in principle observe the uses of script arguments, and include
witnesses in the transaction for just those arguments that are used.
Suitable balancer modifications to achieve this are not part of this
CIP.

### Size costs

Since every part of each running script must either be present in the
transaction, or pulled in via reference inputs, then users will still
have to pay for loading the (transitive) size of the script. However,
if they use reference inputs for some of the arguments then it won’t
all have to be present in the transaction, which reduces the chances
of hitting the size limit.

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

### Cross-module optimisation

Splitting a script into separately compiled parts risks losing
optimisation opportunities that whole-program compilation gives. Note
that script arguments are known in advance, and so potentially some
cross-module optimisation may be possible, but imported modules are
shared subterms between many scripts, and they cannot be modified when
the client script is compiled. Moreover, unrestrained inlining across
module boundaries could result in larger script sizes, and defeat the
purpose of breaking the code into modules in the first place.

This is well-known in the compilers world: separate compilation
reduces the amount of optimization you can do. However, users can
make this tradeoff for themselves, where it makes sense.



## Path to Active
### Acceptance criteria
TODO: The criteria whereby the proposal becomes 'Active'.
### Implementation Plan
TODO: A plan to meet the criteria or N/A
## Versioning
## References
## Appendices
## Acknowledgements

This CIP draws heavily on a design by Michael Peyton Jones.

## Copyright
This CIP is licensed under [CC-BY-4.0]](https://creativecommons.org/licenses/by/4.0/legalcode).
