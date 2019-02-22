# Feature name

* Proposal: [SE-NNNN](NNNN-extract-where-rangereplaceablecollection.md)
* Authors: [Frederick Kellison-Linn](https://github.com/jumhyn)
* Review Manager: TBD
* Status: **Awaiting implementation**

*During the review process, add the following fields as needed:*

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)

## Introduction

When removing elements from a collection, it's often desireable to retrieve the elements that were removed for further processing. Currently, `RangeReplaceableCollection` offers this functionality for methods like `removeFirst()`, `removeLast()`, and `removeAt()`, but bulk removal functions (such as `removeAll(where:)`) don't return anything at all, requiring the user to copy the elements out of the collection beforehand if they want to save them.

The proposed `extractAll(where:)` method would fill this hole, by removing the desired elements and returning a collection of the elements that were removed.

Swift-evolution thread: [Add `extract(where:)` method to `RangeReplaceableCollection`](https://forums.swift.org/t/add-extract-where-method-to-rangereplaceablecollection/20745)

## Motivation

The pattern of removing some item or items from a collection and retaining them for further processing is not uncommon. If we imagine a hypothetical situation where we have a queue of `Task`s that periodically become ready for execution, a method which grabs the ready `Task`s and executes them one at a time, we could implement that today as:

```swift
var readyTasks: [Task] = []
taskList.removeAll(where: { 
    let isReady = $0.isReady 
    if isReady { readyTasks.append($0) }
    return isReady 
}) 
readyTasks.forEach { $0.execute() }
```

Other solutions are either not quite equivalent, require implementing fairly complex algorithms correctly, or are just not that readable:

```swift
// Less readable, and requires a *stable* partition algorithm if order is important
let n = taskList.partition { $0.isReady }
let readyTasks = taskList[n...]
taskList.removeSubrange(n...)
readyTasks.forEach { $0.execute() }

// If isReady is an expensive-to-compute property, this is not optimal
let readyTasks = taskList.filter { $0.isReady }
taskList.removeAll(where: { $0.isReady })
readyTasks.forEach { $0.execute }

// Does not allow us to process readyTasks in any other order (e.g. expected completion time)
// Also, executes the tasks before they are actually removed from the collection
readyTasks.removeAll(where: {
    if $0.isReady {
        $0.execute()
        return true
    } else {
        return false
    }
})
```

## Proposed solution

Using the proposed API, the solution is far more concise and readable:

```swift
let readyTasks = taskList.extractAll(where: { $0.isReady })
readyTasks.forEach { $0.execute }
```

## Detailed design

Add the following method to `RangeReplaceableCollection`

```swift
  /// Removes and returns all the elements that satisfy the given predicate.
  ///
  /// Use this method in place of `removeAll(where:)` to retain the removed
  /// elements for further processing.
  ///
  ///     var phrase = "The rain in Spain stays mainly in the plain."
  ///
  ///     let vowels: Set<Character> = ["a", "e", "i", "o", "u"]
  ///     let extracted = phrase.removeAll(where: { vowels.contains($0) })
  ///     // phrase == "Th rn n Spn stys mnly n th pln."
  ///     // extracted == "eaiiaiaaioeai"
  ///
  /// - Parameter shouldBeExtracted: A closure that takes an element of the
  ///   sequence as its argument and returns a Boolean value indicating
  ///   whether the element should be extracted from the collection.
  ///
  /// - Complexity: O(*n*), where *n* is the length of the collection.
  @inlinable
  public mutating func extractAll(
      where shouldBeExtracted: (Element) throws -> Bool
  ) rethrows {
    let extracted = try Self(self.lazy.filter { try shouldBeExtracted($0) })
    try self.removeAll(where: shouldBeExtracted)
    return extracted
  }
```

## Source compatibility

This change is purely additive.

## Effect on ABI stability

None.

## Effect on API resilience

None.

## Alternatives considered

### Naming

Other potential names were brought up in discussion, but `extractAll(where:)` seemed to be the most favored. Other suggestions included `filterOut(where:)`, `discard(where:)`, and `extract(where:)`.

### Return type

Initial discussions had the return type as `[Self.Element]` rather than `Self`. Since there was no real reason that the return type had to be an array, this proposal opts to just return the type of the `Collection` being operated on.

### Overloading or modifying `removeAll(where:)`

Using a different name for this API is unfortunate since some `remove...` methods already return the removed elements, which could lead to confusion. Thus, another possibility would be to change `removeAll(where:)` to be an `@discardableResult` function which always returns the removed elements. This was avoided since it would mean that even in the cases where the removed items are not needed, `removeAll(where:)` would allocate the memory to store the removed items.

Overloading `removeAll(where:)` was also avoided since it would be a return-type overload which could be confusing and make the API more difficult to use.

