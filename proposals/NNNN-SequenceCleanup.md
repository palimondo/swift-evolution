# Sequence Cleanup

* Proposal: [SE-NNNN](NNNN-SequenceCleanup.md)
* Authors: [Pavol Vaskovic](https://github.com/palimondo), [Author 2](https://github.com/swiftdev)
* Review Manager: TBD
* Status: **Pitch**

## Introduction

This proposal is to change the default implementation of `protocol Sequence` methods so that all operations are eager (not lazy) and return `Array<Element>`. This removes the need to return type erased `AnySequence` that negatively impacts performance in order to hide the heterogenous implementation and provides Swift users with more homogenous interface. The lazy computation of `prefix` and `dropFirst` methods will be moved to `LazySequence`.

Swift-evolution thread: [Discussion thread topic for that proposal](https://lists.swift.org/pipermail/swift-evolution/)

## Motivation

The API of `Sequence` protocol evolved over time to include irregular mixture of eager and lazy methods. In particular, it included lazy implementation of `prefix` and `dropFirst` from before the lazy sequence views found their home in `LazySequence` protocol. They are implemented by internal classes that wrap the underlying sequence and delay the computation until its elements are requested through iteration. [SE-0045](SE-0045.md) added third internal class to implement `drop(while:_)`, that eagerly drops the required element and wraps the partially used `Iterator` from the original sequence. 

All other default implementations of `Sequence` methods are realized on top of `Array`s and perform their computation eagerly. Due to the requirements of the associated type `SubSequence` returned from many `Sequence` protocol methods, all these heterogeneous implementations need to be hidden using type erasure by wrapping the results in `AnySequence`. The heterogeneity of implementation is currently well documented, if the user notices the different complexity of the methods that return lazy wrappers -- Complexity: O(1), instead of O(n). But given the existence of other lazy sequence views on `LazySequence` one might be surprised by this irregularity and overlook this historical quirk of implementation.

Furthermore, the presence of `AnySequence` and `AnyIterator` it returns, present serious optimization obstacle for the Swift compiler, when we chain the calls to these sequence methods. This results is non-specialized generic iterator, that ends up in a tight for-in loop. Currently there is no `prefix` implementation on `LazySequence` - preventing the workaround for avoiding `AnySequence`  by going through `.lazy` call.

```Swift
s.lazy.prefix(maxIterations).prefix(while:{ ... })
```

## Proposed solution

We should take this opportunity to clean up the interface of default `Sequence` implementation to present Swift users with homogenous interface of eager methods that directly return `Array` as `SubSequence`. We'll move the lazy implementation for `prefix` and `dropFirst` to `LazySequence`.

## Detailed design
The following changes will be made to the standard library:
```Swift
extension Sequence {

  public func suffix(_ maxLength: Int) -> [Iterator.Element] { ... }
  
  public func split(
    maxSplits: Int = Int.max,
    omittingEmptySubsequences: Bool = true,
    whereSeparator isSeparator: (Iterator.Element) throws -> Bool
  ) rethrows -> [[Iterator.Element]] { ... }
}

extension Sequence where Iterator.Element : Equatable {
  public func split(
    separator: Iterator.Element,
    maxSplits: Int = Int.max,
    omittingEmptySubsequences: Bool = true
  ) -> [[Iterator.Element]] { ... }
}

extension Sequence where
  SubSequence : Sequence,
  SubSequence.Iterator.Element == Iterator.Element,
  SubSequence.SubSequence == SubSequence {

  /// Returns a subsequence containing all but the given number of initial
  /// elements.
  ...  
  /// - Complexity: O(*n*), where *n* is the length of the sequence.
  @_inlineable
  public func dropFirst(_ n: Int) -> [Iterator.Element] { ... }

  /// Returns a subsequence containing all but the given number of final
  /// elements.
  ...
  /// - Complexity: O(*n*), where *n* is the length of the sequence.
  @_inlineable
  public func dropLast(_ n: Int) -> [Iterator.Element] { ... }
  
  /// Returns a subsequence by skipping the initial, consecutive elements that
  /// satisfy the given predicate.
  ...
  /// - Complexity: O(*n*), where *n* is the length of the collection.
  /// - SeeAlso: `prefix(while:)`
  @_inlineable
  public func drop(
    while predicate: (Iterator.Element) throws -> Bool
  ) rethrows -> [Iterator.Element] { ... }

  /// Returns a subsequence, up to the specified maximum length, containing the
  /// initial elements of the sequence.
  ...
  /// - Complexity: O(*n*), where *n* is the length of the sequence.
  @_inlineable
  public func prefix(_ maxLength: Int) -> [Iterator.Element] { ... }
  
  /// Returns a subsequence containing the initial, consecutive elements that
  /// satisfy the given predicate.
  ...
  /// - Complexity: O(*n*), where *n* is the length of the collection.
  /// - SeeAlso: `drop(while:)`
  @_inlineable
  public func prefix(
    while predicate: (Iterator.Element) throws -> Bool
  ) rethrows -> [Iterator.Element] { ... }
}

// TODO describe lazy views for prefix and dropFirst
```

## Source compatibility

Relative to the Swift 3 evolution process, the source compatibility
requirements for Swift 4 are *much* more stringent: we should only
break source compatibility if the Swift 3 constructs were actively
harmful in some way, the volume of affected Swift 3 code is relatively
small, and we can provide source compatibility (in Swift 3
compatibility mode) and migration.

Will existing correct Swift 3 or Swift 4 applications stop compiling
due to this change? Will applications still compile but produce
different behavior than they used to? If "yes" to either of these, is
it possible for the Swift 4 compiler to accept the old syntax in its
Swift 3 compatibility mode? Is it possible to automatically migrate
from the old syntax to the new syntax? Can Swift applications be
written in a common subset that works both with Swift 3 and Swift 4 to
aid in migration?

## Effect on ABI stability

Does the proposal change the ABI of existing language features? The
ABI comprises all aspects of the code generation model and interaction
with the Swift runtime, including such things as calling conventions,
the layout of data types, and the behavior of dynamic features in the
language (reflection, dynamic dispatch, dynamic casting via `as?`,
etc.). Purely syntactic changes rarely change existing ABI. Additive
features may extend the ABI but, unless they extend some fundamental
runtime behavior (such as the aforementioned dynamic features), they
won't change the existing ABI.

Features that don't change the existing ABI are considered out of
scope for [Swift 4 stage 1](README.md). However, additive features
that would reshape the standard library in a way that changes its ABI,
such as [where clauses for associated
types](https://github.com/apple/swift-evolution/blob/master/proposals/0142-associated-types-constraints.md),
can be in scope. If this proposal could be used to improve the
standard library in ways that would affect its ABI, describe them
here.

## Effect on API resilience

API resilience describes the changes one can make to a public API
without breaking its ABI. Does this proposal introduce features that
would become part of a public API? If so, what kinds of changes can be
made without breaking ABI? Can this feature be added/removed without
breaking ABI? For more information about the resilience model, see the
[library evolution
document](https://github.com/apple/swift/blob/master/docs/LibraryEvolution.rst)
in the Swift repository.

## Alternatives considered

Describe alternative approaches to addressing the same problem, and
why you chose this approach instead.
