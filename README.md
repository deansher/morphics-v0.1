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

... what's different about morphics ...

This implementation of morphics is in PureScript. To help make our explanation simpler
by making it more concrete, this documentation is written in terms of the PureScript
implementation. The underlying ideas are not implementation-specific.

A morphic component is called an "imp". (The name is a pun on "implementation"
and demon.) Each imp forms the root component (potentially the sole component) of
a corresponding data type, which is called the imp's "face". (The name is a pun on human face and "interface".) It is common for multiple imps to represent the same face.

A tree or subtree of imps is called a "clan". A clan is a complete implementation
of its root imp's face. This face is also considered to be the clan's face.
A clan's face can be as simple as `Number`, in which case the clan is as simple as the number `42`. A face is described at runtime by a "meta-face", which is implemented by
a value of the record type `MetaFace`.

A clan is constructed from a "charter", which is JSON. This can be a JSON object,
array, or value, and is represented in PureScript as a `Foreign` value. The module that
supports a given face usually provides a function that takes a charter argument
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
  charter = toForeign 42
  answer :: Number
  answer = foundNumber charter
  log $ "Hello, World! The answer turns out to be " <> show answer <> "."
```

Ok, that was silly -- and yet, complete. Let's move on to a more representative use case.

Suppose we have a face that is a user-defined data type:

```PureScript
type ItemOrder = Item -> Item -> Ordering
```

Suppose we want to have three implementations of `ItemOrder`:
* `orderBySize :: ItemOrder`
* `orderByWeight :: ItemOrder`
* `orderByBlend :: Number -> Number -> ItemOrder` orders by `w * weight + s * size`

So the charter for an `ItemOrder` needs to provide two pieces of information:
* It needs to indicate which of the three functions to choose.
* In the case of `orderByBlend`, it needs to supply `w` and `s`.

An imp is described at runtime by a "meta-imp", which is represented
in PureScript by a value of the record type `MetaImp`.

So, in our `ItemOrder` example, the charter needs to specify an imp of the `ItemOrder` face.
We accomplish this by giving the charter an "imp" property:

```PureScript
bySizeCharter = { imp: "ItemOrder.orderBySizeLabel" }
```

The PureScript implementation of morphics requires each imp to have a label,
which is accessed through the `label` field `MetaImp` record type.
Each imp label should be prefixed by the name of the module in which instances
of that imp are created. In our example, we assume that the module is called "ItemOrder".

In the charter for "bySizeCharter", we use the word "Label" in "orderBySizeLabel" to avoid giving
the impression that the label magically refers to the function name. Although PureScript's
JavaScript runtime would make such magic possible, the morphics implementation uses no
such magic: labels are just strings that are compared at runtime. In practice, a much more
likely choice of label for the `orderBySize` imp would be simply "ItemOrder.orderBySize".

The `orderByBlend` imp adds an interesting new twist: it has parameters `w` and `s`.
Although the parameters in this particular example are simply numbers, they could
have any face. They could be represented by complex clans of their own.

Imp parameters such as `w` and `s` are called "roles". Each meta-imp declares the
roles required by that imp. These are exposed through the `roles` field of the `MetaImp`
record type: `roles :: Array Role`. Just as imps have labels, roles and faces
also have labels. The role labels in our example are "w" and "s".
`Role` is a simple record type:

```PureScript
type Role = { label :: String, faceLabel :: String }
```

As a founder function constructs a clan, it fills each role with a "sept" -- a subordinate clan.
It is up to the founder function how to capture the imp's septs and make them available
to the imp for its operation.

Although it is up to the founder function how to interpret charter JSON and create a clan,
the normal convention is for the charter to be a JSON object with "imp" and "septs" fields.
For example, the following JSON would be a reasonable charter for our `orderByBlend` imp:

```JSON
{
  "imp": "ItemOrder.orderByBlend",
  "septs": {
    "s": 0.7,
    "w": 0.3
  }
}
```

It is up to each face's `MetaFace` instance how an imp should be constructed from the charter JSON.
Here again, there is a common convention. For any given face, such as our `ItemOrder`, the module
implementing that face builds a map from imp label to corresponding imp founder function. If the
PureScript type for the face is `FaceType`, then each imp founder function has the type `Foreign -> FaceType`,
which maps a JSON charter (represented in PureScript as `Foreign`) to a value of `FaceType`.
So the module implementing the face builds a `StrMap (Foreign -> FaceType)`. The `Morphics` module
provides a suitable type alias:

```PureScript
type ImpFounderMap face = StrMap (Foreign -> face)
```

The module that provides support for a face (the "face module") usually provides a function to obtain its
`MetaFace` instance. This function takes an `ImpFounderMap` parameter in case the caller knows imps of that
face that might not be known to the face module, but that should be supported by the face module's founder
function. Similarly, a module that implements one or more imps of a particular face generally exports an
`ImpFounderMap` value that maps imp label to imp founder function for those imps.

If our `ItemOrder` face were implemented according to this convention, and if it followed the conventional
naming pattern, it would supply the following function:

```PureScript
itemOrderMetaFace :: ImpFounderMap -> MetaFace
```

In our example, we have assumed that the `ItemOrder` module implements both the face and all imps
that it knows about. So it needs no special protocol for collecting the imps it knows about: it can
simply hardwire them into `itemOrderMetaFace`, or however else it chooses. But if there were a separate
module (we might imagine `CostSensitiveItemOrders`) that provided additional imps of `ItemOrder`,
then by convention it would export the following value:

```PureScript
itemOrderImps :: ImpFounderMap
```

This value could be used by the `ItemOrder` module if it knows about the `CostSensitiveItemOrders`
module. It could also be used by the caller of `ItemOrder.itemOrderMetaFace` if that caller wanted to
make sure the `CostSensitiveItemOrders` imps were supported, and didn't know whether the `ItemOrder`
module supported them directly.
