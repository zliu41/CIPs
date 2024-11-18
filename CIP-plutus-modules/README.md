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
be used.

This is not to say that code is not shared. On the contrary, there is an
open source repo on `github` called `OpenZeppelin` which appears to be heavily
used. It provides 264 Solidity files, in which 43 libraries are
declared (almost all internal). It seems, thus, that libraries are not
the main way of reusing code in Solidity; rather it is by calling, or
inheriting from, another contract, that code reuse primarily occurs.

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
if the total contract code is still relatively small.

### Static vs Dynamic Linking

With the introduction of modules, scripts will no longer be
self-contained--they may depend on imported modules. Likewise,
modules stored on the blockchain may also depend on imported
modules. An important question is when the identity of those modules
is decided. The actual modules used will need to be supplied in the
transaction running the script--including all the dependencies--but
should the transaction be able to choose modules freely, or should the
choice be constrained using a hash? Another way to phrase this
question is: should the hash of a script depend on the hashes of its
dependencies, or not?

The argument for allowing the calling transaction to supply modules
freely is that it could, for example, supply later versions of some of
the modules used, perhaps fixing bugs. The argument against is that it
weakens security: should a transaction spending a UTxO protected by a
script be able to replace freely some of the code in the verifier? We
think the answer is obviously 'no'!  Allowing it would open for a
variety of clever attacks; allowing *some* module replacements, but
not others, would require mechanisms for validating "correct
upgrades". Such mechanisms ought to be provided via contracts, not
built into the chain. Thus we prefer "static linking", in the sense
that a deployed script (or module) contains the hashes of its
dependencies, and can only be invoked if *the same versions* of those
dependencies are supplied in the transaction; dependency upgrades must
be handled by contracts instead.

Fixing the versions of script dependencies also greatly simplifies
testing the script concerned, or proving it correct. Allowing
different versions of dependencies to be provided when the script is
run would oblige us to clarify what assumptions the script may make
about the imported code, and how those assumptions are to be
guaranteed when implementations of the dependencies are provided.

A related question is whether *all* modules that a script depends on
must be supplied by a calling transaction, or only those that are
actually used in that particular script execution. Allowing the latter
could in some cases dramatically reduce the amount of code that must
be loaded to run a script. For example, we can imagine a verifier that
accepts two different encryption methods, provided by different
modules. In any one attempt to spend the UTxO concerned only one
encryption method would be used, allowing the code from the other to
be omitted. This kind of thing seems to occur occasionally, although
there is no clear evidence that supporting it would enable big savings
in practice.

We refer to this mechanism as "lazy loading". This CIP shows how to
implement it, but also includes a slightly different design that does
not (and achieves some small savings elsewhere as a result).

### Cross-language calls

This CIP does not restrict module interfaces in any way; a module is
just a script computing a value. Some compilers might implement
modules as tuples of exported functions; others might implement them
as functions from function name hashes to functions (this would
resemble the Ethereum mechanism). This CIP provides a low-level way
for modules compiled with different compilers to refer to one another,
but does not otherwise support interlanguage working. It is the job of
compiler writers to provide a foreign function interface, or library
authors to implement their libraries in such a way that they are
usable from other languages.

## Specification

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
The arguments of a script are the imported modules; we need to resolve
them before being able to use them:
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
must provide the pre-image in the usual way.

The only scripts that can be run are complete scripts, so the type of
runScript changes to take a CompleteScript instead of a Script.

### Variation 1

With this design, all module dependencies must be provided for
`resolveScriptDependencies` to succeed. To permit some modules to be
omitted, just replace `traverse` by `map` and change the types
accordingly:
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
        lookupArg :: Arg -> Maybe CompleteScript 
        lookupArg (ScriptArg hash) = do
          script <- lookup hash preimages
          pure $ go script
```
Here `applyScript` is used with the type `CompleteScript -> [Maybe
CompleteScript] -> CompleteScript`; the `Maybe` values should be
encoded using sums-of-products as `constr 0` (for `Nothing`) and
`constr 1 s` for `Just s`. This allows the script to detect missing
modules; the expectation is that scripts will just extract each module
they use from the `Just` value at the point of use, and that this will
fail if the value is `Nothing`.

The cost of this design is an extra step each time a module is used;
the benefit is that fewer modules need be supplied in a transaction,
if some of the imported modules are not needed.

### Variation 2

In this second variation, modules are classified as either
*always-used*, or *sometimes-used*. We replace the `Arg` type by
```
data Arg = AlwaysUsed ScriptHash | SometimesUsed ScriptHash
```
and update `resolveScriptDependencies` as follows:
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
        lookupArg (AlwaysUsed hash) = do
          script <- lookup hash preimages
          go script
	lookupArg (SometimesUsed hash) =
	  case lookup hash preimages of
	    Nothing -> pure $ ...representation of constr 0...
	    Just s -> do s' <- go s
	                 ... representation of constr 1 s' ...
```
Always-used arguments must be supplied in the transaction, and are
passed directly to the script, while sometimes-used ones may be
omitted, and are passed to the script wrapped in a `Maybe` value to
indicate whether they were present or not.

The cost of this variation is a slightly bulkier representation of
scripts stored on the chain; the benefit is that use of 'always
needed' modules is as cheap as in the first variation.

## Rationale: how does this CIP achieve its goals?

Note that mutually recursive modules are not possible.

## Path to Active
### Acceptance criteria
TODO: The criteria whereby the proposal becomes 'Active'.
### Implementation Plan
TODO: A plan to meet the criteria or N/A
## Versioning
## References
## Appendices
## Acknowledgements
## Copyright
This CIP is licensed under [CC-BY-4.0]](https://creativecommons.org/licenses/by/4.0/legalcode).
