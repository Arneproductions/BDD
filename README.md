[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
![Build Status](https://github.com/microsoft/DecisionDiagrams/actions/workflows/dotnet.yml/badge.svg)
![badge](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/rabeckett/4e0516e3b9f171c6b195744513656d58/raw/code-coverage-dd.json)

# Introduction 
This project is an implementation for various variants of [binary decision diagrams](https://en.wikipedia.org/wiki/Binary_decision_diagram) that is used at Microsoft Research. It focuses on high performance, usability, and correctness. The library maintains 100% test coverage.

# Installation
Just add the project to your visual studio solution or add the package from [nuget](https://www.nuget.org/packages/DecisionDiagrams).

# Getting Started
To import the library, add the following line to your file:

```csharp
using DecisionDiagrams;
```

A simple use of the library is shown shown below:

```csharp
// create a manager that uses chain-reduced binary decision diagrams
var manager = new DDManager<CBDDNode>(new CBDDNodeFactory());

// alternatively using traditional BDDs
var manager2 = new DDManager<BDDNode>(new BDDNodeFactory());

// allocate three variables, two booleans and one 32-bit integer
// the internal ordering will match the order allocated from the manager.
var a = manager.CreateBool();
var b = manager.CreateBool();
var c = manager.CreateInt32();

// build formulas from the variables.
DD f1 = manager.Or(a.Id(), b.Id());
DD f2 = manager.And(c.GreaterOrEqual(1), c.LessOrEqual(4));

// get a satisfying assignment for a formula
var assignment = manager.Sat(manager.And(f1, f2));

// get the values as C# objects
bool valuea = assignment.Get(a);  // valuea = false
bool valueb = assignment.Get(b);  // valueb = true
int valuec = assignment.Get(c);   // valuec = 1
```

You can find more detailed examples in the tests.

# Implementation
The library is based on the cache-optimized implementation of decision diagrams [here](https://research.ibm.com/haifa/projects/verification/SixthSense/papers/bdd_iwls_01.pdf), and implements two variants: 
- Binary decision diagrams ([link](https://en.wikipedia.org/wiki/Binary_decision_diagram))
- Chain-reduced binary decision diagrams ([link](https://link.springer.com/content/pdf/10.1007%2F978-3-319-89960-2_5.pdf))

### Data representation
Internally decision diagram nodes are represented using integer ids that are bit-packed with other metadata such as a garbage collection mark bit, and a complemented bit. User references to nodes (`DD` type) are maintained through a separate (smaller) table.

### Garbage collection
The `DD` reference table uses `WeakReference` wrappers to integrate with the .NET garbage collector. This means that users of the library do not need to perform any reference counting, which is common in BDD libraries. Nodes are kept in a memory pool and the library maintains the invariant that a node allocated before another will appear earlier in this pool. This allows for various optimizations when looking up nodes in the unique table. To uphold this invariant, the library implements a mark, sweep, and shift garbage collector that compacts nodes when necessary. 

### Memory allocation
By hashconsing nodes in the unique table, the library ensures that two boolean functions are equal if and only if their pointers (indices) are equal. The unique table holds all nodes and is periodically resized when out of memory. For performance reasons, we ensure that this table is always a power of two size. This makes allocating new space a bit inflexible (harder to use all memory) but in return makes all operations faster. To compensate for this inflexible allocation scheme, the library becomes more reluctant to resize the table as the number of nodes grows.

### Optimizations
The library makes use of "complement edges" (a single bit packed into the node id), which determines whether the formula represented by the node is negated. This ensures that all negation operations take constant time and also reduces memory consumption since a formula and its negation share the same representation. The implementation also includes a compressed node type `CBDDNode` that should offer lower memory use and often higher performance but comes with the restriction that you can not create more than 2^15-1 binary variables.

### Operations
Internally, the manager supports several operations: conjunction, existential quantification, if-then-else and then leverages free negation to support other operations efficiently. It also leverages commutativity of conjunction + disjunction to further reduce memory by ordering the arguments to avoid redundant entries. Currently, the library does not support dynamic variable reordering as well as a number of operations such as functional composition.

# Performance

The performance of the library should be comparable to other highly optimized BDD implementations. Below are the timings to solve the famous n-queens chess problem (how to arrange n queens on an n x n chess board such that none attack each other). The library is compared to BuDDy, which is considered to be one of the fastest BDD implementations, as well as JavaBDD, which has a direct translation of the C-based BuDDy implementation into Java. The times given are using .net core 6.0 for a 64-bit Intel Core i7-8650U CPU @ 1.90GHz machine. All implementations require around 200MB of memory, while the CBDDNode implementation uses roughly half that at 100MB of memory.

| Implementation               | Language | n  | Time (seconds) |
| ---------------------------- | -------- | -- | -------------- |
| DecisionDiagrams (CBDDNode)  | C#       | 12 | 11.4s          |
| DecisionDiagrams (BDDNode)   | C#       | 12 | 14.8s          |
| BuDDy (aggressive allocation)| C        | 12 | 21.9s          |
| JavaBDD (BuDDy translation)  | Java     | 12 | 27.5s          |
| BuDDy (default settings)     | C        | 12 | 35.9s          |

# Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.
