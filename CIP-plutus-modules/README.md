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
self-contained--they will depend on imported modules. Likewise,
modules stored on the blockchain will also depend on imported
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
different version of dependencies to be provided when the script is
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

## Specification
## Rationale: how does this CIP achieve its goals?
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
