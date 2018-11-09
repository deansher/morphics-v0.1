_Deprecated: replaced by [deansher/morphics](https://github.com/deansher/morphics)_

# Morphics (Vaporware)

Morphics is a new way of writing software that enables new kinds of code reuse.
It allows programmers, non-programmers and machine-learning algorithms to
assemble modules in novel but meaningful ways. We gain these benefits by adding
a new dimension for software decomposition, which we call "behavior
decomposition". This is fundamentally different from the mainstream approach to
decomposing software, which, in contrast, we call "dependency decomposition". A
morphic program or module is decomposed along both dimensions: it still uses
APIs from "dependency" libraries like mainstream software, but it also uses
subordinate behaviors.

Morphics allows us to write software modules that define spaces of possible
behaviors instead of just single, specific behaviors. This is more difficult for
a module author because it requires thinking more abstractly about what the
module might do. As programmers, we are already used to implementing lower-level
abstractions that behave sensibly when recombined in novel ways by higher-level
code. Now, in morphics, we also expect ourselves to implement higher-level
abstractions that behave sensibly in the face of novel behavior by subordinate
libraries. In return, we get modules that can be assembled into a broad range of
behaviors.

We can picture behavior decomposition as top-down templating. A parent module
acts as a template. At an abstract level, it implements a particular behavior,
but it deliberately leaves details of that behavior open to be specified by
subordinate modules. When we assemble a complete tree of morphic modules, this
pins down all behavioral details and creates a runnable program.

The power of morphics comes from our ability to make larger or smaller
adjustments to the behavior of a morphic program or module by swapping out
modules higher in the tree or deeper in the tree. Since subordinate modules
refine the behavior of parent modules, a module replacement deep in the tree
usually yields a modest change in overall behavior.

Any given collection of morphic modules (not yet assembled into a program)
defines an abstract, high-level language that expresses a range of possible
program or module behaviors. This language is not brittle, in the sense that
there are simply defined syntactic changes (replacing modules deep in the tree)
that result in small, sensible changes to program behavior. The range of
behaviors defined by a collection of morphic modules is dense in useful
behaviors and relatively easy to understand, so it  can be efficiently explored
by non-programming end users or by machine learning.

## Contrasting "Behavior Decomposition" with "Dependency Decomposition"

When we write a software module with behavior decomposition, we implement an
abstract behavior, while deliberately leaving room for this behavior to be
refined by a later choice of subordinate modules. For example, we might
implement a parent module that defines full-text search as an abstract behavior.
We could define a plug-in point for a subordinate module that interprets the
text string, plus another plug-in point for a subordinate module that searches a
particular text database. When we define a plug-in point, we specify an
interface that the subordinate module must implement. Any given plug-in
interface is commonly implemented by multiple subordinate modules that provide
alternative behaviors for the interface. Subordinate modules can define their
own plug-in points, so the modules form a tree of behaviors.

Each subordinate module refines its parent's behavior: when we install a
subordinate module into a parent's plug-in point, the resulting combination of
modules has a more concrete behavior than the parent alone. If we fill out an
entire tree of subordinate modules -- by starting with one module and then
recursively filling in subordinate modules for every plug-in point -- the
completed module tree has concrete behavior. It can be executed. It does one
particular thing.

In contrast, let's think about what happens when we decompose our code along the
familiar "dependency" axis. In that case, we think of a parent module's code as
expressing one particular behavior. We intend it to do one particular thing.
When we use APIs from subordinate modules, we make assumptions about the
behavior of those APIs. We may worry about whether we fully understand the
specified or de facto behavior of the subordinate API. We may worry about
whether we are correctly using that API, and whether the API is itself
implemented correctly. We typically think of the API as being defined by
the subordinate module -- or at least defined independently as a standard --
rather than being defined by the parent module.

When we use "dependency decomposition", we know that our subordinate APIs hide
details. We know we are giving them latitude to implement these details in
different ways. We sometimes use portable APIs that we expect will be
implemented in very different ways on different platforms. But we write our
higher-level code with the belief that it fully specifies our program's
behavior down to whatever level of detail we care about.

When we use "behavior decomposition", we approach things differently. Each
parent module defines a space of possible behaviors. It deliberately leaves its
own behavior open-ended -- defining plug-in points with interfaces that can have
multiple alternative implementations to further specify behavior.

## The Features and Terminology of Morphics

Morphics has other unique or uncommon features. Generally, these features
help put the "top-down templating" character of behavior decomposition into
practice:

* A morphic super-behavior routes all of its interactions with any particular
  subbehavior through a single API data type. (This API data type is called a "face".
  Morphic behaviors are called "imps".) Other data types may be involved in
  the interaction, but these data types are associated with the face -- the API --
  and are independent of the imp -- the implementation.

* The wiring of super-imps to sub-imps is entirely described in JSON. So
  super-imps never make compile-time assumptions that they are using particular
  sub-imps. The wiring can be changed by changing JSON "charters", or at
  runtime.

* Both faces and imps export runtime interfaces that describe themselves both
  for human consumption and for machine consumption. So tooling can automate the
  wiring. At one extreme, there can be a UI for interactively snapping together
  imps into a module or a program. At the other extreme, programs can search
  through the space of legal wirings, much like genetic algorithms.

The value proposition of morphics is that these three unusual characteristics,
when used together and applied consistently, enable a world of snap-together behaviors that is
easy to understand, easy to work with, and provides a rare level of code reuse.

## Morphic Programming

This implementation of morphics is in PureScript. To help make our explanation simpler
by making it more concrete, the rest of this documentation is written in terms of the
PureScript implementation. The underlying ideas are not implementation-specific.

A morphic behavior is called an "imp". (The name is a pun on "implementation"
and demon.) Each imp forms the root behavior (potentially the sole behavior) of
a corresponding data type, which is called the imp's "face". (The name is a pun on human
face and "interface".) It is common for multiple imps to represent the same face.

A tree or subtree of imps is called a "clan". A clan is a complete implementation
of its root imp's face. This face is also considered to be the clan's face.
A clan's face can be as simple as `Number`, in which case the clan is as simple as
the number `42`. A face is described at runtime by a "meta-face", which is implemented by
a value of the record type `MetaFace`.

A clan is constructed from a "charter", which is JSON. This can be a JSON object,
array, or value, and is represented in PureScript as a `Foreign` value. The module that
supports a given face must provide a function that takes a charter argument
and returns an instance of the face. This is called the "founder function" for
the face. By convention, if the face is represented as a PureScript type called
`FaceType`, the founder function is called `foundFaceType`.

Within an imp's PureScript code, simple data types like `Number` are
most commonly handled in the normal PureScript fashion. In this case, a 42 is just a 42.
But when a simple data type is used as a configurable parameter that adjusts the behavior
of a clan, it is handled as a face and becomes part of the morphic hierarchy.
So let's go ahead and use our `Number` example to make these ideas more concrete.

To create a `Number` clan, we first need an appropriate charter.
We also need a founder function for `Number`. We might write something like this:

```PureScript
module ObtainAnswer where

import Prelude
import Data.Foreign (toForeign)
import Control.Monad.Eff.Console (log)

import Morphics.Number (foundNumber)

main = do
  charter = toForeign {
    imp: "Morphics.Number.number"
    data: 42
  }
  answer :: Number
  answer = foundNumber charter
  log $ "Hello, World! The answer turns out to be " <> show answer <> "."
```

Ok, that was silly -- and yet, complete. Let's move on to a more representative use case.

Suppose we have a face that is the following PureScript data type:

```PureScript
type ItemOrder = Item -> Item -> Ordering
```

Suppose also that we want to have three implementations of `ItemOrder`:
* `orderBySize :: ItemOrder`
* `orderByWeight :: ItemOrder`
* `orderByBlend :: Number -> Number -> ItemOrder` orders by `w * weight + s * size`
Each implementation is simply a PureScript function. We will encapsulate each function
in an imp. An imp is described at runtime by a "meta-imp", which is represented
in PureScript by a value of the record type `MetaImp`.

The charter for an `ItemOrder` needs to provide two pieces of information:
* It needs to indicate which imp to use.
* In the case of `orderByBlend`, it needs to supply `w` and `s`.

Charter JSON has an "imp" property that specifies an imp by giving its label:
```PureScript
bySizeCharter = { imp: "ItemOrder.orderBySizeLabel" }
```

An imp's label is defined through the `label` field of the `MetaImp` record type.
By convention, each imp label is prefixed by the name of the
module in which instances of that imp are created. In our example, we assume that the
module is called "ItemOrder".

In our `bySizeCharter` example, we gratuitously use the word "Label" in "orderBySizeLabel"
to avoid giving the impression that the label magically refers to the function name.
Although PureScript's JavaScript runtime would make such magic possible, the morphics
implementation uses no such magic: labels are just strings that are compared at runtime.
In practice, a much more likely choice of label for the `orderBySize` imp would be
simply "ItemOrder.orderBySize".

The `orderByBlend` imp adds an interesting new twist: it has parameters `w` and `s`.
Such imp parameters are called "roles". Although in this particular example they
are simply numbers, they could have any face. They could be implemented by complex
clans of their own.

Each meta-imp declares the roles required by that imp. These are exposed through the
`roles` field of the `MetaImp` record type: `roles :: Array Role`. Just as imps have labels, roles and faces
also have labels. The role labels in our example are "w" and "s". `Role` is a simple record type:
```PureScript
type Role = { label :: String, faceLabel :: String }
```

As a founder function constructs a clan, it implements each role with a "sept" -- a subordinate clan.
It is up to the founder function how to capture the imp's septs and make them available
to the imp for its operation.

An imp is represented in charter JSON as an object with three fields:
* imp: the label of the imp
* roles (optional): an object where the keys are role labels and the values are sept charters
* data (optional): additional data for use by the imp's founder function in constructing the imp

For example, the following JSON would be a reasonable charter for our `orderByBlend` imp:
```JSON
{
  "imp": "ItemOrder.orderByBlend",
  "roles": {
    "s": { "imp": "Morphics.Number.number", "data": 0.7 },
    "w": { "imp": "Morphics.Number.number", "data": 0.3 }
  }
}
```

To support the implementation of face and imp founder functions,
the `Morphics` package provides a `MorphicSpace` monad, with the following operations:

```PureScript
-- Provide a MetaImp (which contains the imp's name).
-- It is fine to register multiple `MetaImp`s for the same imp name and
-- face type, so long as those `MetaImp`s are Eq.
registerImp  :: forall face. MetaImp face -> MorphicSpace Unit

-- Provide a MetaFace.
-- It is fine to register multiple `MetaFace`s for the same
-- face type, so long as those `MetaFace`s are Eq.
registerFace :: forall face. MetaFace face -> MorphicSpace Unit

-- Obtain a MetaImp for the given imp name and face type.
-- If no such MetaImp has been registered, the computation continues through its "error check only" flow.
getImp  :: forall face. String -> MorphicSpace (MetaImp face)

-- Obtain a MetaFace for the given face type.
-- If no such MetaImp has been registered, the computation continues through its "error check only" flow.
getFace :: forall face. MorphicSpace (MetaFace face)

-- Record an error.
appendError :: String -> MorphicSpace Unit

-- Switch to this monad's "error check only" flow.
errorCheckOnly :: forall a. MorphicSpace a
```

Any module that provides imps or faces must export a function with the following
name and signature:
```PureScript
registerMorphicElements :: MorphicSpace Unit
```

Any module that depends on other modules that may want to provide imps or faces
must invoke the `registerMorphicElements` functions of those modules from its own
`registerMorphicElements` function.
