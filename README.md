# Morphics

Morphics is a different approach to program construction.
It is based on hierarchical assembly of components.

The idea behind morphics is that it is both beneficial and feasible
to construct substantial software modules entirely from hierarchical,
snap-together components. Each component is implemented with traditional code.
Each higher-level component specifies (in both machine-readable and human-readable form)
the interfaces it needs from its subordinate components. There can be multiple
implementations of each component interface.

If you stand back far enough, this is how we already construct most modern
programs and libraries. Each program or library depends on a well-defined set
of dependencies -- libraries with well-defined interfaces. These subordinate
libraries are versioned and managed through build tools. So what's different
about morphics?

The following characteristics of the caller/library relationship are each
rare outside morphics, but guaranteed in morphics:

* A morphic super-component routes all of its interactions with any particular
  subcomponent through a single API data type. (This API data type is called a "face".
  Morphic components are called "imps".) Other data types may be involved in
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
when used together and applied consistently, enable a world of snap-together components that is
easy to understand, easy to work with, and provides a rare level of code reuse.

This implementation of morphics is in PureScript. To help make our explanation simpler
by making it more concrete, the rest of this documentation is written in terms of the
PureScript implementation. The underlying ideas are not implementation-specific.

A morphic component is called an "imp". (The name is a pun on "implementation"
and demon.) Each imp forms the root component (potentially the sole component) of
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
