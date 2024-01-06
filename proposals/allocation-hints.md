
---
layout: doc-page
title: "Object allocation hints"
author: Wojciech Mazur
---

**By: Wojciech Mazur**

## History

| Date          | Version            |
|---------------|--------------------|
| Jan 06th 2022 | Initial Draft      |

## Summary
Allow to annotate class constructors with a hint how object should be allocated: on the heap using GC, on the stack or via dedicated arena allocators (`Zone`s).

## Motivation
Due to the object-oriented nature of Scala, one of the most frequent operations done by runtime is the allocation of new objects. Currently, all allocated objects are by default allocated and managed by the GC. The only exception from that is an experimental allocation of classes using `scala.scalanative.memory.SafeZone` by specifying explicitly or implicitly an arena allocator used to allocate and free required memory. It is powered by Dotty Capture Calculus (project Caprese) allowing it to provide type-safety over allocated memory, especially it allows to detect at compile time illegal memory state, espcially use-after-free scenarios. Unfortunetlly, it would take an underdetermined amount of time until this feature would become stable and publicly available to users in stable Scala versions.

## Proposed solution
Until typesafe memory management for allocated objects is available, users might be tempted to manually allocate and manage allocated objects. Unfortunately, currently, there is no available syntax allowing to allocation of classes on the stack or in the unsafe memory zone. One of the reasons for that was the lack of good representation of handles to class constructors as they're strongly entangled with a `new` operator. One workaround would be to use a dedicated `apply` method, but it would require a large amount of boilerplate and would not allow to work with 3rd-party types, e.g. Java standard library. 

To solve this issue we propose annotation-based hints for the compiler which would modify the Scala Native backend to emit custom allocators for objects. Thanks to the usage of annotations we can provide stubs for other platforms (JVM, Scala.js) allowing for 0-zero abstraction and easy cross-compilation.
We propose to provide initially 3 allocation hints, informing the compiler that should given type be allocated on the stack, on the heap using GC, or in an available memory zone provided by the user.

The annotations can be only applied to local `val`/`var` statements or as the type annotation of an expression. It would be illegal to annotate a member of a class - their allocation is predefined by the caller of class constructor.

Custom allocations not involving GC are possible only because Scala Native has never introduced object finalizers - since we don't need to track which object would be collected by the GC and potentially require to run finalization code, we can safely allocate them on the stack or in the explicit zone allowing for the most optimal freeing of resources. 

## API
User user-facing part of the interface involves only a set of annotations triggering special handling in the Scala Native compiler plugin.

```scala
import scala.annotation.meta
import scala.scalanative.unsafe.Zone

@meta.field
sealed abstract class allocationHint extends scala.annotation.StaticAnnotation

object allocationHint {
  // Would allocate annotated objects on the stack
  final class stack extends allocationHint

  // would allocate annotated object using GC (default allocation type), can override allocations in blocks
  final class gc extends allocationHint

  // would allocate annotated object using memory zone
  final class zone(implicit zone: Zone) extends allocationHint
}

```

## Example of usage
```scala
import scala.scalanative.annotation.allocationHint._
import scala.scalanative.unsafe.Zone

object Examples {
  def localValues() = {  
    // would allocate x on stack
    @stack val x = new String("foo")

    // would allocate y using gc
    @gc val y = new {}

    // would allocate z using implicit or explicit unsafe.Zone (or memory.SafeZone under Dotty with Capture Calculus enabled)
    given Zone = ???
    @zone val z1 = new {}
    @zone(using summon[Zone]) val z2 = new {}

    // Allocate every instance in rhs on the stack with exception of `x2` which would be allocated using GC
    @stack val block = {
      val x1 = new {}
      @gc val x2 = new {}
      (x1, x2)
    }
  }

  def anonymousInstance() = {
    // We cannot add annotations to expressions, use type annotation instead eg. `<expr>: @stack`
    // Allocate object on stack and pass it to function
    println(new {}: @stack)

    locally {
      val x1 = new {}
      @gc val x2 = new {}
      (x1, x2)
    }: @stack
  }
}

object InvalidUsage {
  // class members cannot have defined allocation hint
  @stack val forbidden = new String("forbidden") // compile-error

  locally {
    // Multiple hints create ambiguity
    @stack @gc val ambiguous = new {} // compile-error
  }
}

```

The draft of the solution for annotation hints is available in https://github.com/WojciechMazur/scala-native/tree/feature/class-allocation-hints

## Interaction between custom allocators and GC
One of the most important aspects of providing a safe execution of programs using custom allocation hints would be runtime safety. We need to ensure that objects are allocated on the stack, but having fields possibly referring to objects managed by the GC would be reachable by the GC while scanning.

In case of allocation on the stack, the Scala Native needs no to little amount of modifications to make it safe when working with GC. Currently, all GC implementations are already scanning the stack of each thread. If an object would be allocated on the stack, we would always reach its inner fields when scanning.

However, custom allocation zones are not scanned by default - these allocators can be using any arbitrary chunk of memory internally. To make their usage safe we would first need to inform the GC about custom roots for scanning. This strategy was already successfully applied to `scala.scalanative.memory.SafeZone` and would not require a big amount of changes for other `scala.scalanative.unsafe.Zone` implementations.

## In-direct object allocations
In Scala code we often use companion objects `apply` methods to allocate new instances of objects. It's especially useful when we need to execute some logic before calling an object constructur. We don't want to introduce any special handling for these kind of operations. Instead we would fallback to Scala 3 inlining mechanism. After the Scala 3 compiler `inlining` phase the inlined right-hand expression of the annotated expression would becoume indistinguishable for the Scala Native backend allowing to apply annotation hints of the caller expression.

## Opportunities
By providing an alternative, user-defined memory management over-allocated objects we might provide an opportunity to lower GC bottlenecks and time spent on garbage collection. It might be beneficial for memory-space costly functions or areas of code that could be manually tuned for best performance. 

## Risks
One of the biggest risks of the new feature would be allowing for the introduction of illegal memory states in user programs. Especially use-after-free errors when storing stack/zone-allocated memory in GC-allocated objects or returning it the the caller of the function. Some of these runtime errors could be detected at compile time, either in the Scala compiler or after Scala Native optimizers (especially after inlining) to check if memory allocated on the stack is returned to the function caller, which might lead to undefined behavior at runtime. Without the support of compile-time allocation capabilities checks of Dotty Capture Calcusus (project Caprese), we would never have been able to successfully detect all of the memory management issues.

## Compatibility
The changes would require to include additional information to the NIR format about the allocation hints. These can be introduced in a backward compatible way to the NIR format, in the attached initial implementation we introduce a "normal" `nir.Op.{Class,Array}Alloc` instruction which don't have store any information in NIR about hints, and the alternative op-code for less-likely hinted allocation. The binary and source compatibility of Scala/JVM `nir` definitions would be broken as they're defined using case classes. We don't plan to introduce complexity of toolchain by introducing alternative `nir.Op` instructions.

