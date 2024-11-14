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
created by one transaction, which is subject to the overall
transaction size limit. Competing blockchains suffer from no such
limit: on the Ethereum chain, contracts can call one another, and so
the code executed in one transaction may come from many different
contracts, created independently on the blockchain--each subject to a
contract size limit, but together potentially many times that
size. This enables more sophisticated contracts to be implemented;
conversely, on Cardano, it is rather impractical to implement
higher-level abstractions as libraries, because doing so will likely
exceed the script size limit. This is not just a theoretical problem:
complaints about the script size limit are the most common complaint
made by Cardano contract developers.

Thus the primary goal of this CIP is to lift the limit on the total
amount of code run during a script execution, by allowing part of the
code to be provided in external modules. By storing these modules on
the blockchain and providing them as reference UTxOs, it will be
possible to keep transactions small even though they invoke a large
volume of code.

Once scripts can be split into separate modules, then the question
immediately arises of whether the script and the modules it imports
need to be in the same language or not. Today there are many languages
that compile to UPLC, and then run on the Cardano blockchain. Ideally
it should be possible to define a useful library in *one* of these
languages, and then use it from all of them. A secondary goal is thus
to define a module system which permits this, by supporting
cross-language calls.

### The Situation on Ethereum

Ethereum contracts are not directly comparable to Cardano scripts;
they correspond to both the *on-chain* and the *off-chain* parts of
Cardano contracts, so one should expect Ethereum contracts to require
more code for the same task, since in Cardano only the verifiers need
to run on the chain itself.

Nevertheless, it is interesting to ask how the need for modules has
been met in the Ethereum context.

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
