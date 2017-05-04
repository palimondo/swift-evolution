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

Proposed changes are fully source compatible with Swift 3.

## Effect on ABI stability and resilience

The change does effect ABI and should be implemented before we freeze the ABI.

## Alternatives considered

Alternatively we could do nothing and carry the historical baggage in the Sequence API forever, as we freeze the ABI.
