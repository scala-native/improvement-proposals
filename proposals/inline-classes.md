---
layout: doc-page
title: "Inline Classes"
author: Martin Odersky
---

# Pre SNIP: Inline Classes

Object-oriented and functional architectures encourage the creation of many heap values with pervasive sharing. For instance,
in Scala and Java the list `List(1, 2, 3)` would generate 6 heap objects: the three numbers are _boxed_ in `Integer` objects
which then are chained in the `Cons` nodes. We can use a "flatter" representation by e.g. using a `Vector` instead of a `List`.
But even then the numbers would be boxed.

Another facet of this problem is presented by custom classes. Say we have
```scala
class Color(rgb: Int)
class Complex(re: Double, im: Double)
class Point(coord: Complex, color: Color)
```
Then a `Point` consists of three objects or types `Point`, `Complex` and `Color`. One might have wished instead for a single object where
the `coord` and `color` fields are inlined.

This note describes a representation that
allows inlining the fields of specially marked _inline_ classes everywhere values of such classes are used. At the same time, it supports generics in a way that is efficient in both speed and code size, without needing compile-time or link-time monomorphization or pervasive boxing. (The scheme does need some monomorphizing fixups at runtime, and does box in some situations, which are relatively rare.)

The techniques described here require a processor architecture with 64 bit pointers.

## Defining Inline Classes

An inline class is defined by adding the modifier `inline` to a class definition. E.g. the following definitions would make the classes above inline classes.
```scala
inline class Color(rgb: Int)
inline class Complex(re: Double, im: Double)
inline class Point(coord: Complex, color: Color)
```
Inline classes are implicitly `sealed`. They cannot have inner classes.
If they have child classes, these must be inline classes as well.
For now, we also require that they only have immutable fields and that they don't have type parameters or type members.

Primitive classes such as `Int`, `Double` or `Boolean` all count as inline classes.

Inline classes must directly or indirectly extend `AnyVal`. If no explicit class parent is listed in an inline class definition, `AnyVal` is assumed. In a sense,  inline classes can be understood as version 2 of Scala's value classes. They drop several of the restrictions of value classes, in particular, they can have more than one field. On the other hand, they introduce the new restriction that inline classes cannot be parameterized. Parameterized value classes are currently allowed but they are constrained by a number of technical restrictions that makes the feature less usable than one might expect.

## Representation of Inline Classes

As the name implies, values of inline classes are inlined with all their fields in any object that refers to them.
For instance, the `Point` class would have three fields:

 - Two `Double` fields, which come form inlining `coord`
 - An `Int` field, which comes from inlining `color`.

Unlike normal classes, inline classes do not define a header, so the total size of a `Point` would be 8 + 8 + 4 = 20 bytes.

Inline classes determine the alignment of their instances depending on their size.
A typical scheme would align at 8-byte boundaries for sizes >= 8, and round up to the next power of two for sizes lower than 8.

If there is a hierarchy of inline classes (which must all be defined in the same compilation unit), all inline classes are given the same size, which is the size of
the root inline class.

When passing a value of an inline class to a (non-generic) function, the value is passed by reference if its size exceeds a threshold
(typically, the size of a word, i.e. 8 bytes). When returning a value of an inline class with a size over that threshold, we reserve space to hold
the value in the caller, and pass a reference to that space to the callee. The callee then copies the result value to the reserved space.

References to inline class values follow a stack-discipline. Since heap-allocated objects always contain copies of inline class values instead of references to them, it is not possible for a reference to a stack-allocated inline class value to exceed the lifetime of the frame in which the value is defined.

A possible loophole to this rule would be where we upcast a value of an inline class to `AnyVal` or `Any` or to a universal trait.
In that case, we need to convert the reference to a regular pointer (which cannot point to the stack). We do this by _boxing_,
which means creating a normal stand-alone object. The class info of that stand-alone object would be a normal class that mirrors the data and the behavior of the inline class. Alternatively, we could define a boxed inline class as a regular class that contains the inline
class value as its only field and forwards all methods defined by the inline class to that field.

We will see later that boxing is not required when passing an inline class value to a generic context; only monomorphic upcasts require boxing. We would therefore expect boxing to be relatively rare.

## Representation of Inline Class References

Given an inline class reference, we need to be able to access not just its fields but also the class info and vtable of the inline class. Since inline classes don't come with a header the only way to store this info is in the reference itself. Here is a scheme how this can be accomplished in 64 bits. Our representation is such that a full address range of 48 bits is supported (which is the maximum virtual address range of current processor architectures). At the same time, it allows for close to 448K different inline classes. We assume here that all heap pointers that go to full objects are aligned at 16 byte boundaries whereas
alignments of inline references follow their natural sizes.

An inline class reference is composed of two fields, listed from high to low bits:

 - an address field of 45 bits,
 - a vtable index of 19 bits

The vtable index could be implemented in one of two ways:

 - as an index into an array of vtable references, or
 - as a shifted pointer directly to the class info of an inline class (which would be aligned at some coarse granularity).

Table indices ending in `0000` binary are reserved; they cannot be used for inline classes. This means that regular pointers and inline references can be distinguished according to whether the low 4 bits of their address are 0 or not.

We can convert an inline class reference to a machine address as follows

 1. Define the raw address `ra = r >>> 19 << 3`.
 2. From the vtable index `i` in `r`, compute the `alignment` of the inline class
     1. If `alignment == 8`, the class fields are 8-byte
        aligned, and their start address is `ra`.
     2. If `alignment == 4`, the class fields are 4-byte aligned, and
        their start address is `ra + (i & 4)`. In this case there are
        2 identical entries for the inline class in the `vtable` array
        which are at locations `i` and `i +/- 4`. Bit 2 of the index of the entry tells us the low 4 bytes of the address.
     3. Analogously, if the `aligment == 2`, the class fields are 2-byte aligned, and their start address is
        `ra + (i & 6)`. In this case there are
        4 identical entries for the inline class in the `vtable` array at positions spaced 2 apart.
     4. Analogously, if the `aligment == 1`, there are 8 entries for the inline class at consecutive positions in the `vtable` array and the address of the class field is `ra + (i & 7)`.

Note that alignments less than 8 only arise for inline class references used in some generic context. The general conversion from references to machine addresses
is therefore relatively rare; in most cases inline references _are_ machine addresses. The conversion can be expressed in pseudo-code as follows:

    def toAddr(r: InlineReference): Address =
      val vi = r & 0x7FFFF  // mask bits 0..18
      val vt = vtable(vi)
      val ra = r >>> 19 << 3
      ra + (vi & vt.MASK)

where `vt.MASK` is defined depending on the alignment of the class
described by `vt`.

  |Alignment|MASK|
  |---------|----|
  |        8|   0|
  |        4|   1|
  |        2|   3|
  |        1|   7|

When going from an inline class reference arising from a generic context to some data fitting in an 8 byte word in a monomorphic context,
we need to perform the same conversion, but then `vt.MASK` is statically known, so no vtable lookup is needed and the computation can be simplified.

## Generics

Implementations of generics fall into two main camps:

 - **Monomorphization**, which means that type parameters are expanded at compile-time or link-time, leading to code that is specialized with the actual type arguments.
 - **Boxing**, which means data accessed from a generic context uses a uniform  representation (typically an object accessed via a pointer). Data have to be
 converted in and out of that representation, which is typically achieved by constructing
 and deconstructing object wrappers.

Monomorphization optimizes for speed at the expense of code size, compile times and binary modularity. Some kinds of Monomorphization impose restrictions on expressiveness in order to avoid infinite expansions of certain recursive types (for instance, it would be challenging to express a `zip` method with full naive monomorphization).

Boxing optimizes for code size and compilation simplicity and speed, but it incurs overheads at run time.

Languages using boxing include Eiffel, OCaml, Haskell, Java, and Scala. Languages using monomorphization include C++, Rust, and Go. Blends between the two techniques exist. For instance, .NET languages perform a late monomorphization of primitive type arguments when JIT compiling. This circumvents the challenges that monomorphization presents for stable binary interfaces. Or, Swift monomorphizes
inside a module but uses uniform "resilient" representations of data in public APIs.

Over the last 20 years the shifts in processor performance favored monomorphization, since boxing introduces indirections and indirections became increasingly costly.

It turns out that inline classes and their references permit an interesting combination of very lightweight monomorphization with a uniform representation that does not require boxing. This combination combines some of the advantages of both schemes.

The idea is that, as for boxing, we use a uniform representation of generically accessed data. That representation is a disjoint union type
of either a regular pointer or an inline class reference. Regular objects are passed
with regular pointers and values of inline classes are passed by reference. So values of inline classes do not have to be boxed to objects. As shown when we discussed inline
class references, the low-endian bits of a word determine
whether it is a pointer or an inline class reference.

Since primitive types are inline classes, instead of boxing a primitive value we now simply make sure it is addressable (by storing it in a local variable if needed), and pass
the reference to that variable. Instead of unboxing a primitive value we simply dereference that inline class reference.

## Example

As an example, consider how generic function types are implemented for primitive operations. Here is the
trait `Function1` in Scala:
```scala
  trait Function1[-T, +R]:
    def apply(x: T): R
```
With the new generics representations, this trait would erase to
```scala
  trait Function1:
    def apply(x: Addr, result: Addr): Unit
```
Here, `Addr` is name of the aforementioned disjoint union type of inline class references and pointers.

An instance such as `(x: Int) => x + 1` would translate (at least conceptually) to a subclass of `Function1`:
```scala
new Function1[Int, Int]:
  def apply(x: Int): Int = x + 1
```
This implementation would erase to
```scala
new Function1:
  def apply(x: Int): Int = x + 1
  override def apply(x: Addr, result: Addr): Unit =
    result.asIntRef.set(this.apply(x.asIntRef.get))
```
Here, the second `apply` is the usual bridge method generated by erasure. The `asIntRef` method
casts an `Addr` to an inline class reference of `Int`. We know it is that statically, so no runtime
test is needed and the following expansion applies:
```scala
    a.asIntRef  =  (a >>> 19 << 3) | (a & 4)
```
The `get` method on such references loads the referenced `Int` value whereas the
`set` method stores an `Int` value at the referenced address.

## Monomorphization

Two other problems need to be solved. First, we need to be able to invoke virtual methods defined in the upper bound of a type variable. To do this, we can access the vtable defined in the reference.

Second, we need special provisions when creating generic objects. Let's say we want to create a pair object, i.e. an instance of a class like this one:
```scala
class Pair[A, B](fst: A, snd: B)
```
Each type parameter could refer to an inline class or a regular class. If type parameter `A` is instantiated with
a regular class, it is a normal pointer, and that pointer will be stored in the `fst` field of class `Pair`.
But if `A` is an inline class, we need to copy all fields of that class
to corresponding fields in `Pair`. This also means that the start address of the `snd` element of `Pair` will depend
on the size of that inline class.

We achieve this customization through lightweight monomorphization. The idea is that we create on demand a _class instance info_ for each
unique instantiation of a generic class with inline classes. For instance, there would be such an info generated
for `Pair[Color, Int]`. Since inline classes are not generic, instantiation can be done in a single step. A class could have multiple infos,
for each combination of classes in its type parameters, where all regular classes count as one. So, for instance
`Pair[String, String]` would have the same info as `Pair[Pair[Int, Int], List[Int]` (assuming `Pair`, `String` and `List` are all regular classes). But `Pair[Int, Boolean]`, `Pair[Int, String]`, or `Pair[Int, Int]` would have different infos. Infos for the same class share many components, including the
vtable. They differ in two items that need to be added per info:

 1. A field for the total size of the instance class.
 2. For each field that has the type of a type parameter: a way to compute a reference to the field. This could
   be stored as an offset to be added to the (possibly shifted) address of the enclosing object. The offset would contain both the
   field offset and the vtable info for the inline class. If the last four bits of this offset are `0000` this would indicate that the field is of a regular class type. In that case, the computed value needs to be dereferenced once to get
   the correct reference.

Note that we need computed offsets only for generic fields.
We can choose a layout where non-generic fields
come before generic ones, so non-generic fields have fixed offsets that do not
depend on instance types.

**Example**:

For `Pair[Color, Int]`, the `fst` parameter info would be the offset
```scala
  (v-table-index of Color)
```
and the `snd` parameter info would be the offset
```scala
  (size-of-Color-in-words <<< 19) + (v-table-index of Int)
```
For `Pair[Color, String]`, the `fst` parameter info is as before and the
`snd` parameter info would be the offset `(size-of-Color)`, rounded up to a multiple of 8.

To compute a reference from a base address `a` and an offset `o`, proceed as follows:
```scala
  if (o & 0xF) == 0 then (a + o)^ else (a <<< 19) + o
```

## Computing ClassInstance Infos

Computing class instance infos involves two tasks.

 1. Find out whether a new instance is needed, or whether the instance was already generated.
 2. If the instance is not yet generated, compute sizes and reference offsets according to the rules given above.

Since the number of computed instances is bounded, the cost of (2) is easily amortized. For (1), one could implement several scenarios.

One scenario would maintain one hash table for each class `C` with a type parameter `X`. The hash table maps inline class arguments `I` for `X` to the addresses of the class instance infos `C[I]`. Classes with more than one type parameter are thought of being in a curried representation. The first argument would then get mapped to a synthetic partially applied class with its own hash table, instead of being mapped to the final class info.

This scheme can be tweaked. For instance, it seems plausible that on average a user-defined inline class occurs as the type of a generic field of relatively few generic classes. So for user-defined inline classes it might be better to search in _their_ class info for generic classes containing _them_ , and this might even be feasible in most instances by a simple scan of a packed array fitting in one cache line. On the other hand, for primitive types we could use an array representation in each generic class containing addresses of vtables for primitive type arguments. E.g., to find out whether `Some[Int]` has been implemented we would index with a fixed offset into the primitive class array of class `Some`. Conversely, to find out whether `Some[Color]` has been implemented we would scan the array of all implemented generic containers of `Color` which would be part of the class info of `Color`. If that array exceeds a certain threshold (typically the size of one or two cache lines, which means 16 to 32 elements) we could fall back to a hash table.

Hash-table lookup might also be sped up by keeping inline caches of recently looked up class infos, exploiting the rule that a few combinations of types will probably make up the majority of allocations. Nevertheless lookup of implemented class infos might represent a significant cost in high-allocation scenarios.

The second scenario achieves faster lookup at the price of a whole program link step. The idea is to compute an envelope of all needed ClassInstance infos at link time, to generate all these classInfos and to use a _row displacement table_ for mapping generic classes and inline arguments to vtable instances (or partial class infos for multi-parameter classes). A row displacement table is essentially a compressed
sparse matrix where all rows are mapped into one large vector where each row starts at a different offset chosen so that elements defined in different rows never get mapped to the same element in the vector. Row displacement tables
are already used in Scala Native and other implementations for trait type tests
and dynamic dispatch of trait methods.

In that scenario a vtable lookup is very fast (just load a pointer for the outer class, add an offset for the inline class and dereference). But it assumes a global
link step. If one admits generic dynamically loaded libraries, the row-displacement table might have to be updated at runtime, which could be expensive.

## Layout of ClassInstance Infos

Class instance infos share large parts of their data with the class info of the generic parent class. The shared parts include the virtual method tables, which can become large. There are several ways to deal with this:

 - Duplicate common data in each class info. Pressure on data space size could be alleviated by sometimes mapping a physical page (containing e.g. a large virtual method table) to several virtual pages forming part of individual class instance infos.
 - Introduce an indirection from a class instance info to the shared parts. This saves space at the cost of an extra load to access these parts.
 - If enough bits are available in the object header layout: Include references
 to both the shared and the individual part of a class instance info.


## Mutable Variables and Lazy Vals

Mutable variables and lazy vals of generic type pose the problem that sometimes space needs to be allocated without having an initial value that describes the  class. Here are two examples:

```scala
lazy val x: T = ...
var y: T | Null = null
```
If a `ClassTag[T]` or a value of type `T` is in scope, we can obtain the class instance info for `T`. If `T` extends `AnyRef`, we know `T` cannot be an inline class, and therefore the var or val holds a pointer. If neither is the case, we have two choices:

 1. When storing a value in a generic mutable variable or initializing a generic lazy val, box the value if it is an inline class reference.
 2. Or, refuse to compile the program and demand that a `ClassTag` is provided for `T`.

We can also implement a combination of both approaches: Box every assigned inline class value, but warn at compile time that this is an inefficiency that can
be avoided by providing a `ClassTag`.

## Arrays

Arrays support inline classes in unboxed form. This is easy since array creation demands a `ClassTag` already. If the `ClassTag` refers to an inline class, each element
of the array will contain the fields of the class directly. The size of such an element is then the padded size of the inline class. Padding means: If the size of the inline class is less than 8, the size is rounded up to the next power of two.
If the size is at least 8, it is rounded up to the next multiple of 8.

## Garbage Collection Considerations

One new situation with inline classes is that we can now have inline class references that point to the middle of objects. These references can only live
on the stack, not in the heap.

This means root marking needs to be adapted. If we use conservative root marking by simply scanning stacks for possible candidates, we need to be able anyway to
identify where heap objects start. It should be easy to adapt such as scheme to
recognize inline class references as well. If an inline class reference pointing to the heap is found, we need to mark transitively all its embedded pointers as roots, and we need to mark the enclosing object as alive. On the other hand, there is no need to mark any pointers outside the embedded inline class field as roots.

A different scheme suggests itself if precise stackmaps are used to identify roots, and if at the same time the heap layout makes it hard to identify starts of objects from pointers to the middle of them. We would then still mark embedded pointers of inline references as roots, as before. But instead of identifying the enclosing objects of such inline references, we could record all the references in a separate buffer. Once that's done, convert the buffer to a set with efficient range search (e.g. sort and do lookup by binary search). Then, when it comes to freeing or copying an object, we would consult that set for inline references pointing to the middle of that object. If any exist,
they would block freeing the object and they would have to be adjusted if the object is copied. Inline class references pointing to the heap can only exist in
generic contexts, so the number of references to record is probably reasonably small.

## Benefits and Costs

The scheme described here beings several benefits over the previous erasure by boxing scheme, including

 - better data locality through inline classes,
 - less boxing of primitive types,
 - fewer objects to allocate,
 - generic data and specialized data have the same representation, which should make it simpler to specialize.

The main costs and disadvantages are due to the following factors.

 - More vtable space is needed to account for class instance infos of parametric classes.
 - Sharing of components of class instance infos might need an additional indirection, depending on the chosen ClassInstance layout.
 - Identifying previously computed class instance infos represents an additional cost for allocation of objects of parameterized classes.
 - Some additional complications might arise for GC.
 - It would be difficult to implement the techniques if usable address space needs to be increased significantly beyond 48 bits.


