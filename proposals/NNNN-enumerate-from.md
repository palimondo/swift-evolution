# Feature name

* Proposal: [SE-NNNN](NNNN-enumerate-from.md)
* Authors: [Pavol Vaskovic](https://github.com/palimondo), [Author 2](https://github.com/swiftdev)
* Review Manager: TBD
* Status: **Pitch**

* Bugs: [SR-4746](https://bugs.swift.org/browse/SR-4746)

## Introduction

Let user specify the staring index for `enumerated` method on `Sequence`s.

Swift-evolution thread: [Discussion thread topic for that proposal](https://lists.swift.org/pipermail/swift-evolution/)

## Motivation

The enumerated() method defined in an extension on protocol Sequence always counts from 0.
When you need the numbers to be counting up from different index, you have to post process 
the resulting tuple in an inconvenient way.

## Proposed solution

We could provide an option to count elements from a user specified offset:
```swift
[6, 7, 8].enumerated(from: 6)
// [(offset: 6, element: 6), (offset: 7, element: 7), (offset: 8, element: 8)]
```
If implemented with default parameter, this does not change the usage for existing code, 
being source compatible with Swift 3. 

## Detailed design

The proposed solution is to propagate the starting value to the internal counter on 
`EnumeratedIterator` and set the default starting value to 0.

```swift
public struct EnumeratedIterator<
    Base : IteratorProtocol
> : IteratorProtocol, Sequence {
    internal var _base: Base
    internal var _count: Int
    
    /// Construct from a `Base` iterator.
    internal init(_base: Base, _offset: Int) {
        self._base = _base
        self._count = _offset
    }
    
    /// The type of element returned by `next()`.
    public typealias Element = (offset: Int, element: Base.Element)
    
    /// Advances to the next element and returns it, or `nil` if no next element
    /// exists.
    ///
    /// Once `nil` has been returned, all subsequent calls return `nil`.
    public mutating func next() -> Element? {
        guard let b = _base.next() else { return nil }
        let result = (offset: _count, element: b)
        _count += 1
        return result
    }
}

public struct EnumeratedSequence<Base : Sequence> : Sequence {
    internal var _base: Base
    internal let _offset: Int
    
    /// Construct from a `Base` sequence.
    internal init(_base: Base, _offset: Int) {
        self._base = _base
        self._offset = _offset
    }
    
    /// Returns an iterator over the elements of this sequence.
    public func makeIterator() -> _EnumeratedIterator<Base.Iterator> {
        return EnumeratedIterator(_base: _base.makeIterator(), _offset: _offset)
    }
}

extension Sequence {
    public func enumerated(from: Int = 0) -> _numeratedSequence<Self> {
        return EnumeratedSequence(_base: self, _offset: from)
    }
}
```

## Source compatibility

Proposed change is source compatible with Swift 3.

## Effect on ABI stability and resilience

This change does affect the ABI and should be implemented before we freeze it.

## Alternatives considered

Currently proposed workaround for the lack of flexibility in `enumerated()` is to 
use `zip` with the collection and half-open range. From [SR-0172 One-sided Ranges](https://github.com/apple/swift-evolution/blob/master/proposals/0172-one-sided-ranges.md):
> Additionally, when the index is a countable type, `i...` should form a `Sequence` 
> that counts up from `i` indefinitely. This is useful in forming variants of 
> `Sequence.enumerated()` when you either want them non-zero-based i.e. 
> ` zip(1..., greeting)`, or want to flip the order i.e. `zip(greeting, 0...)`.
Drawback of that approach is you need to use free function `zip`, forcing a break in 
the chain of sequence operations, as there is currently no `zipped` method on `Sequence`.

If this is the preffered approach, we should consider removing the `enumerated()` method
altogether, as the limited usefullness in its current state hardly justifies the space 
on API surface it occupies.
