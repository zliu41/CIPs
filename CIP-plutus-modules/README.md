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
witness in the invoking transaction, or via a [reference script](https://cips.cardano.org/cip/CIP-0033) in that
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

#### Variation: Lazy Loading

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

#### Variation: Value Scripts

In this variation, script code is syntactically restricted to explicit
λ-expressions with one λ per `ScriptArg`, followed by a syntactic
value. (Values are constants, variables, built-ins, λ-abstractions,
delayed terms, and SoP constructors whose fields are also values).

This means that every script must take the form
`λA1.λA2....λAn.<value>`, where `n` is the number of `ScriptArg`s
supplied. Now, since variables in `CompiledCode` are de Bruijn indices
then the `n` λs can be omitted from the representation--we know how
many there must be from the number of `ScriptArg`s, and the names
themselves can be reconstructed.

`Script`s in this restricted form can be mapped directly into CEK
values, without any CEK-machine evaluation steps. In pseudocode:
```
scriptCekValue
  :: Map ScriptHash CekValue
  -> Script
  -> CekValue
scriptCekValue scriptValues (ScriptWithArgs head args) =
  cekValue (Env.fromList [case lookup h scriptValues of
	   		    Just v -> v
			    Nothing -> vbuiltin unit
	   		 | ScriptArg h <- args])
	   (deserialize (getPlc head))
           
```
That is, a script is turned into a value by creating a CEK machine
environment from the values of the `ScriptArg`s, and converting the
body of the script (a syntactic value) in a CekValue in that
environment.

This pseudocode follows the 'lazy loading' variation; an easy
variation treats not finding a script hash as an error.

Syntactic values are turned into `CekValue`s by the following
function, which is derived by simplifying `computeCek` in
UntypedPlutusCore.Evaluation.Machine.Cek.Internal, and restricting it
to syntactic values.
```
cekValue
  :: Env
  -> NTerm
  -> CEKValue
cekValue env t = case t of
  Var _ varname      -> lookupVarName varName env
  Constant _ val     -> VCon val
  LamAbs _ name body -> VLamAbs namea body env
  Delay _ body       -> VDelay body env
  Builtin _ bn       ->
    let meaning = lookupBuiltin bn ?cekRuntime in
    VBuiltin bn (Builtin () bn) meaning
  Constr _ i es      ->
    VConstr i (foldr ConsStack EmptyStack (map (cekValue env) es)
  _                  -> error
```
Converting a syntactic value to a CekValue does require traversing it,
but the traversal stops at λs and delays, so will normally traverse
only the top levels of a term.

Finally, if `preimages` is the `Map ScriptHash Script` constructed from
a transaction, then we may define
```
scriptValues = Map.map (scriptCekValue scriptValues) preimages
```
to compute the CekValue of each script.


Scripts are then applied to their arguments by building an initial CEK
machine configuration applying the script value to its argument value.

Note that this recursive definition of `scriptValues` could potentially allow an
attacker to cause a black-hole exception in the transaction validator,
by submitting a transaction containing scripts with a dependency
cycle. However, since scripts are referred to by hashes, then
constructing such a transaction would require an attack on the hash
function itself... for example a script hash `h` and values for `head`
and `args` such that
```
h = hash (Script head (h:args))
```
We assume that finding such an `h` is impossible in practice; should
this not be the case, then we must build a script dependency graph for
each transaction and check that it is acyclic before evaluating the
scripts in this way.


##### Cost

Converting `Script`s to `CekValue`s does require a traversal of all
`Script`s, and the top level of each `Script` value. This is linear
time in the total size of the scripts, though, and should be
considerably faster than doing the same evaluation using CEK machine
transitions. The conversion can be done *once* for a whole
transaction, sharing the cost between several scripts if they share
modules (such as frequently used libraries). So costs should be
charged *for the whole transaction*, not per script. The most accurate
cost would be proportional to the total size of values at the
top-level of scripts. A simpler approach would be to charge a cost
proportional to the aggregated size of all scripts, including
reference scripts--although this risks penalizing complex scripts with
a simple API.

##### Implementation concerns
The CEK implementation does not, today, expose an API for starting
evaluation from a given configuration, or constructing `CekValue`s
directly, so this variation does involve significant changes to the
CEK machine itself.

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

newtype ModArg = ModArg ScriptHash

data Mod = forall b. Mod{ modCode :: Maybe (CompiledCode b),
     	   	     	  modArgs :: Maybe ([ModArg]),
			  modHash :: ScriptHash }
```

Here the `modArgs` correspond to the `ScriptArg`s in the UPLC case,
and the `modHash` is the hash of the underlying `Script`.  The type
parameter of `Module a` is a phantom parameter, just like the type
parameter of `CompiledCode a`, which tells us the type of value which
the application of the `modCode` to the `modArgs` represents. 

We can convert any `ScriptHash` into a module:
```
fromScriptHash :: ScriptHash -> Module a
fromScriptHash hash = Module Nothing Nothing hash
```
and we can convert any `CompiledCode` into a module:
```
makeModule :: CompiledCode a -> Module a
makeModule code = Module (Just (Mod code)) (Just []) ...compute the script hash...
```

We also need a way to supply an imported module to a `Module`:
```
applyModule :: Module (a->b) -> Module a -> Module b
applyModule (Module (Mod (Just code) (Just args) _)) m =
  Module (Mod (Just code) (Just (args++[modHash m])) ...compute the script hash...)
```
As in UPLC, the intention is that scripts that import modules be
written as lambda-expressions, and the imported module is then
supplied using `applyModule`. No change is needed in the Plinth
compiler to support this mechanism.

Note that only a `Module` containing code and an argument list can
have the argument list extended by `applyModule`; this is because the
`ScriptHash` of the result depends on the code and the entire list of
arguments, so it cannot be computed for a module that lacks either of
these.

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

The design in this CIP supports both alternatives. Suppose a module
`A` imports modules `B` and `C`. Then module `A` will be represented
as the lambda-expression `λB.λC.A`. This can be compiled into a
`CompleteScript` and placed on the chain, with en empty list of
`ScriptArg`s, as a reference script in a UTxO, allowing it to be used
with any implementations of `B` and `C`--the calling script must pass
implementations of `B` and `C` to the lambda expression, and can
choose them freely. We call this 'dynamic linking', because the
implementation of dependencies may vary from use to use. On the other
hand, if we want to *fix* the versions of `B` and `C` then we can
create a `Script` that applies the same `CompleteScript` to two
`ScriptArg`s, containing the hashes of the intended versions of `B`
and `C`, which will then be supplied by
`resolveScriptDependencies`. We call this 'static linking', because
the version choice for the dependency is fixed by the script. It is up
to script developers (or compiler writers) to decide between static
and dynamic linking in this sense.

On the other hand, when a script is used directly as a validator then
there is no opportunity to supply additional arguments; all modules
used must be supplied as `ScriptArg`s, which means they are
fixed. This makes sense: it would be perverse if a transaction trying
to spend a UTxO protected by a spending validator were allowed to
replace some of the validation code--that would open a real can of
worms, permitting many attacks whenever a script was split over
several modules. With the design in the CIP, it is the script in the
UTxO being spent that determines the module versions to be used, not
the spending transaction. That transaction does need to *supply* all
the modules actually used--including all of their dependencies--but it
cannot choose to supply alternative implementations of them.

### In-service upgrade

Long-lived contracts may need upgrades and bug fixes during their
lifetimes. This need is met on the Ethereum blockchain using the
'proxy pattern'--a 'proxy' contract which delegates calls to the
current implementation contract, whose identity is stored in the proxy
contract's mutable state. Proxy contracts can provide a 'code upgrade'
method which modifies the mutable state to store a new implementation
contract.


The Cardano chain does not offer mutable state. Instead, a changing
state is represented by a succession of UTxOs, each holding the
current state, usually with the currently-valid UTxO identified by
holding a particular NFT. In the absence of mutable state a dependency
cannot be updated just by changing a pointer, but scripts can still be
upgraded by creating new values on the chain. The exact mechanism
depends on the kind of script--and, often, on the original script
developer preparing the ground for a later code change.

Note that, on Ethereum, a proxy contract can be updated without
changing its contract address---thanks to mutable state. On Cardano, a
script address *is* the hash of its code; of course, changing the code
will change the script address. It is very hard to see how that could
possibly be changed without a fundamental redesign of Cardano. So the
methods discussed below are different in nature from the Ethereum one:
they upgrade dependencies in something by replacing it with a new one,
with different dependencies. This is really just functional
programming at work: data is always 'updated' by creating a new
version with possibly different content. This does mean that script
addresses are going to change when their dependencies do; there's no
way around it.

First consider shared modules, stored as reference scripts in
UTxOs. The hash of a module depends on the hash of all its
dependencies, so when a dependency changes, then a new version of the
UTxO needs to be created with the new dependency, and its hash needs
to be distributed (by off-chain means). To prevent accidental use of
the old UTxO, it could be spent.

UTxOs whose *spending verifier* needs upgrading can be spent and
recreated with a new verifier, if the need has been anticipated by the
script author. The verifier would need to accept a 'code change'
redeemer, and then check that the transaction created a new UTxO
protected by the new spending verifier. For example, the code change
redeemer might provide the script hash of the new verifier, and the
old verifier would then check that the new UTxO, with the same
contents, was protected by that hash. This mechanism permits an
arbitrary code change; of course this opens for attacks, so in
practice such a verifier would need to check that the proposed code
change was correctly authorised. How this is done is up to the
contract concerned.

Note that *currency symbols* in Cardano are just the hash of the
minting policy--a script. Thus, updating a dependency of a minting
policy means changing the currency symbol. We need to be able to
convert tokens with the old currency to the new one. To allow this,
the minting script must allow *burning* the old currency when the new
one is being minted, and the new minting script must allow minting
when the old currency is being burned, provided the code upgrade is
correctly authorized. This is enough to allow wallets holding the old
currency to replace it by the new one--a wallet can just submit a
transaction that burns the old tokens and mints the new. If the
currency is also to be stored in UTxOs protected by spending
verifiers, then those verifiers must also accept 'currency upgrade'
redeemers, and check that the UTxO is just being recreated with old
tokens replaced by new ones--or alternatively, continue to use the
old coins and exchange them when they reach a wallet. To facilitate
this, transactions that require an input with these tokens should also
accept old versions, along with authentication of the code upgrade.

Staking validators are a much simpler case: they can be upgraded just by
deregistering the state key registration certificate that refers to
them, and then reregistering the same state key with a new staking
validator.

### Lazy loading

The 'lazy loading' variation in the specification section above
permits fewer modules to be supplied in a transaction.  Dependency
trees have a tendency to grow very large; when one function in a
module uses another module, it becomes a dependency of the entire
module and not just of that function. It is easy to imagine situations
in which a script depends on many modules, but a particular call
requires only a few of them. For example, if a script offers a choice
of protocols for redemption, only one of which is used in a particular
call, then many modules may not actually be needed. The variation
allows a transaction to omit the unused modules in such cases. This
reduces the size of the transaction, which need provide fewer
witnesses, but more importantly it reduces the amount of code which
must be loaded from reference UTxOs.

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

#### Balancer modifications

To take advantage of 'lazy loading', it's necessary to identify
reference scripts that are *dynamically* unused, when the scripts in a
transaction run. The best place to do that is in a transaction
balancer, which needs to run the scripts anyway, both to check that
script validation succeeds, and to determine the number of execution
units needed to run the scripts. We adopt the view that

**A transaction balancer may drop reference inputs from a
   transaction, if the resulting transaction still validates**

We call reference scripts which are not actually invoked during script
verification 'redundant'; these are the reference scripts that can be
removed by the balancer.

##### First approach: search

The simplest way for a balancer to identify redundant reference inputs
is to try rerunning the scripts with an input removed. If script
validation still succeeds, then that input may safely be removed. The
advantages of this approach are its simplicity, and lack of a need for
changes anywhere else in the code. The disadvantage is that
transaction balancing may become much more expensive--quadratic in the
number of scripts, in the worst case.

The reason for this is that removing one script may make others
redundant too; for example if script A depends on script B, then
script B may become redundant only after script A has been
removed--simply evaluating script A may use the value of B, and
scripts are evaluated when they are passed to other scripts, whether
they are redundant or not. So if the balancer tries to remove B first,
then script verification will fail--and so the balancer must try again
to remove B after A has been shown to be redundant. Unless we exploit
information on script dependencies, after one successful script
removal then all the others must be revisited. Hence a quadratic
complexity.

In the case of 'value scripts' this argument does not apply:
evaluating a script will never fail just because a different script is not
present. In this case it would be sufficient to traverse all the
scripts once, resulting in a linear number of transaction
verifications.

##### Second approach: garbage collection

First the balancer analyses all the scripts and reference scripts in a
transaction, and builds a script dependency dag (where a script
depends on its `ScriptArg`s). Call the scripts which are invoked
directly in the transaction (as validators of one sort or another) the
*root* scripts.

Topologically sort the scripts according to the dependency relation;
scripts may depend on scripts later in the order, but not
earlier. Now, traverse the topologically sorted scripts in order. This
guarantees that removing a *later* script in the order does not cause
an *earlier* one to become redundant.

For each script, construct a modified dependency graph by removing the
script concerned, and then 'garbage collecting'... removing all the
scripts that are no longer reachable from a root. Construct a transaction
including only the reference scripts remaining in the graph, and run
script validation. If validation fails, restore the dependency graph
before the modification. If validation succeeds, the script considered
and all the 'garbage' scripts are redundant; continue using the now
smaller dependency graph.

When all scripts have been considered in this way, then the remaining
dependency graph contains all the scripts which are dynamically needed
in this transaction. These are the ones that should be included in the
transaction, either directly or as reference scripts.

The advantage of this approach is that only the code in the balancer
needs to be changed. The disadvantage is that transaction balancing
becomes more expensive: script verification may need to be rerun up to
once per script or reference script. In comparison to the first
approach above, this one is more complex to implement, but replaces a
quadratic algorithm by a linear one.

##### Third approach: modified CEK machine

The most direct way to determine that a script is not redundant is to
observe it being executed during script verification. Unfortunately,
the CEK machine, in its present form, does not make that
possible. Thus an alternative is to *modify the CEK machine* so that a
balancer can observe scripts being executed, and declare all the other
scripts redundant. In comparison to the first two approaches, this is
likely to be much more efficient, because it only requires running
script verification once.

The modifications needed to the CEK machine are as follows:

`CekValue`s are extended with *tagged values*, whose use can be
observed in the result of a run of the machine.
```
data CekValue uni fun ann =
  ...
  | VTag ScriptHash (CekValue uni fun ann)
```
In the 'value script' variation, no expression resulting in a `VTag`
value is needed, because `VTag`s will be inserted only by
`resolveScriptDependencies`. In other variations, a `Tag` constructor
must also be added to the `NTerm` type, to be added by
`resolveScriptDependencies`. In either case the version of
`runScriptDependencies` *used in the balancer* tags each value or
subterm derived from a `ScriptHash` `h` as `VTag h ...` (or `Tag h
...` in variations other than 'value scripts').

The CEK machine is parameterized over an emitter function, used for
logging. We can make use of this to emit `ScriptHash`es as they are
used. This allows the balancer to observe which `ScriptHash`es *were*
used.

Simply evaluating a tagged value, or building it into a
data-structure, does not *use* it in the sense we mean here: replacing
such a value with `builtin unit` will not cause a validation
failure. Only when such a value is actually *used* should we strip the
tag, emit the `ScriptHash` in the `CekM` monad, and continue with the
untagged value. This should be done in `returnCek`, on encountering a
`FrameCases` context for a tagged value, and in `applyEvaluate` when
the function to be applied turns out to be tagged, or when the
argument to a `builtin` turns out to be tagged.

Adding and removing tags must be assigned a zero cost *in the
balancer*, since the intention is that they should not appear in
transactions when they are verified on the chain. Thus a zero cost is
required for the balancer to return accurate costs for script
verification on the chain. On the other hand, if these operations *do*
reach the chain, then they should have a *high* cost, to deter attacks
in which a large number of tagging operations are used to keep a
transaction verifier busy. This can be achieved by adding a `BTag`
step kind to the CEK machine, a `cekTagCost` to the
`CekMachineCostsBase` type, and modifying the balancer to set this
cost to zero during script verification.

The advantage of this approach is that it only requires running each
script once in the balancer, thus reducing the cost of balancing a
transaction, perhaps considerably. The disadvantage is that it
requires extensive modifications to the CEK machine itself, a very
critical part of the Plutus infrastructure.

##### Fourth approach: lazy scripts

TODO: Possibly add a discussion of tracing script use by exploiting
lazy evaluation.

### Value Scripts

This section discusses the 'value scripts' variation.

The main specification in this CIP represents a `Script` that imports
modules as compiled code applied to a list of `ScriptHash`es. To
prepare such a script for running, `resolveScriptDependencies`
replaces each hash by the term it refers to, and builds nested
applications of the compiled code to the arguments. These applications
must be evaluated by the CEK machine *before* the script proper begins
to run. Moreover, each imported module is itself translated into such
a nested application, which must be evaluated before the module is
passed to the client script. In a large module hierarchy this might
cause a considerable overhead before the script proper began to
run. Worst of all, if a module is used *several times* in a module
dependency tree, then it must be evaluated *each time* it is
used. Resolving module dependencies traverses the entire dependency
*tree*, which may be exponentially larger than the dependency *dag*.

The value script variation addresses this problem head on. Scripts are
converted directly into CEK-machine values that can be invoked at low
cost. Each script is converted into a value only once, no matter how
many times it is referred to, saving time and memory when modules
appear several times in a module hierarchy.

On the other hand it does restrict the syntactic form of
scripts. Scripts are restricted to be syntactic lambda expressions,
binding their script arguments at the top-level. This is not so
onerous. But modules must also be syntactic values at the top
level. For example, consider a module represented by a record, whose
fields represent the exports of the module. Then all of those exports
need to be syntactic values--an exported value could not be computed
at run-time, for example using an API exported by another
module. While many module exports are functions, and so naturally
written as λ-expressions (which are values), this restriction will be
onerous at times.

This method does require opening up the API of the CEK machine, so
that CEK values can be constructed in other modules, and introducing a
way to run the machine starting from a given configuration. So it
requires more invasive changes to the code than the main
specification.

### `ScriptHash` allowed in terms?

An alternative design would allow UPLC terms to contain `ScriptHash`es
directly, rather than as λ-abstracted variables, to be looked up in a
global environment at run-time. This would address ths same problem:
the cost of performing many applications before script evaluation
proper begins. This would also require changes to the CEK machine, and
is not really likely to perform better than the 'value scripts'
variation (in practice, the main difference is the use of a global
environment to look up script hashes, as opposed to many per-module
ones). However, this approach is less flexible because it does not
support dynamic linking (see Static vs Dynamic Linking above). Once a
`ScriptHash` is embedded in a term, then a different version of the
script cannot readily be used instead.

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

### Impact on optimisation and script performance

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
algorithms, because their code is smaller, and easier to fit within
the script size limit. Removing the need to make such choices can
potentially improve performance considerably.

### Related work

#### Merkelized Validators

Philip DiSarro describes ["Merkelized
validators"](https://github.com/Anastasia-Labs/design-patterns/blob/main/merkelized-validators/merkelized-validators.md),
which offer a way of offloading individual function calls to stake
validators: the client script just checks that the appropriate stake
validator is invoked with a particular function-argument and result
pair, checks that the argument equals the argument it wants to call
the function with, and then uses the result as the result of the
function. The stake validator inspects the argument-result pair,
computes the function for the given argument, and checks that the
result equals the result in the pair. This design pattern enables the
logic of a script to be split between the client script and the stake
validator, thus circumventing the limits on script size. It also
permits the result of a function to be computed *once*, and shared by
many scripts, thus achieving a kind of memoisation. However, it is a
little intricate to set up, and not really an alternative to a module
system.

#### The WebAssembly Component Model

The [Web Assembly Component
Model](https://component-model.bytecodealliance.org/) defines a
high-level IDL to enable components written in different programming
languages (such as C/C++, C#, Go, Python and Rust), to work together
in one WebAssembly system. WASM already has a module system, and a
WASM component may consist of a number of modules (written in the same
language). The focus here is on interaction between *different*
programming languages in a well-typed way. Defining such an IDL for
Cardano might be useful in the future, but it is too early to do so
now.

### Preferred Options

Allowing script code to be spread across many transactions lifts the
most commonly complained-about restriction faced by Cardano script
developers. It permits more complex applications, and a much heavier
use of libraries to raise the level of abstraction for script
developers. Modules are already available on the Ethereum blockchain,
and quite heavily used. Adopting this CIP, in one of its variations,
will make Cardano considerably more competitive against other smart
contract platforms.

The *main alternative* in this CIP is the simplest design, is easiest
to implement, but suffers from several inefficiencies.

The *lazy loading* variation allows redundant scripts to be omitted
from transactions, potentially making transactions exponentially
cheaper. To take full advantage of it requires a balancer that can
drop redundant scripts from transactions. Three alternative methods
are described: *search*, the simplest, which must run script
verification a quadratic number of times in the number of scripts in
the worst case; *garbage collection*, a self-contained change to the
balancer which analyses script dependencies and thus needs to run script
verification only a linear number of times; a *modified CEK machine*
which adds tagged values to the machine, which the balancer can use to
identify redundant scripts in *one* run of script verification,
possibly requiring one more run to make accurate exunit cost estimates.

The *value scripts* variation restricts scripts to be explicit
λ-expressions binding the script arguments, with an innermost script
body which is a syntactic value. Such scripts can be converted to CEK
values in a single traversal; each script can be converted to a value
*once per transaction*, rather than at every use. This variation is
expected to reduce the start-up costs of running each script
considerably; on the down-side the syntactic restriction would be a
little annoying, and it requires CEK operations which are not
currently part of the API, so it requires modifications to a critical
component of the Plutus implementation.

The simplest alternative to implement would be the main alternative
without variations. The most efficient implementation would combine
value scripts with lazy loading, using tagged values in the CEK
machine to analyse dynamic script dependencies in the balancer, and so
drop redundant scripts from each transaction. However, this requires
modifications to the CEK machine and the balancer as well as resolving
dependencies in scripts.

Our recommendation is: TBD.

## Path to Active
### Acceptance criteria
TODO: The criteria whereby the proposal becomes 'Active'.
### Implementation Plan
TODO: A plan to meet the criteria or N/A
## Categories
This is a Plutus CIP.

As a Plutus CIP, it leaves UPLC, TPLC, PIR and Plinth *unchanged*
except for the addition of an API for building and using modules in
Plinth. In the case of the 'modified CEK machine' alternative for
'balancer modifications', without 'value scripts', it requires *adding
a construct* to UPLC to tag values so that their use can be observed;
this is a *minor* change which is backwards-compatible, but it would
require a new PlutusCore language version.

Because this CIP changes the representation of scripts, it requires a
new Plutus Core ledger language, and can only be introduced at a hard
fork.

As far as the Ledger is concerned, the representation of a script is
just `bytes`. This does not change. Therefore there are no changes to
the Ledger, and thus this is not a Ledger CIP,

## Acknowledgements

This CIP draws heavily on a design by Michael Peyton Jones.

## Copyright
This CIP is licensed under [CC-BY-4.0]](https://creativecommons.org/licenses/by/4.0/legalcode).


