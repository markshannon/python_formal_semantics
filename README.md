# python_formal_semantics

Partial specification and examples for formal specification of Python's semantics.

## Small step operational semantics

We aim to define the semantics of Python using [small step operational semantics](https://en.wikipedia.org/wiki/Operational_semantics#Small-step_semantics).

Conventionally, small step semantics are using defined directly in terms of syntax.
However, given the complex semantics of some statements, such as the `try`-`finally`, and `with` statements, we start by defining the operational semantics of an abstract machine using a set of operations. The semantics of each statement can then be defined by defining a translation from syntactice element to a list of operations.

## The semantics

The semantics consists of interrelated parts.

* A definition of an abstract machine
* A start state for that machine
* A halt state for that machine
* A transition function for that machine

Execution of a program consists of the sequence of transitions that begins at the start state and ends at the halt state.

### Semantics components

* [Top level](.overview.md)
* [Abstract machine](./machine.md)
* [Transition functions](.operations.md)


## Python implementation

As part of the definition we also aim to implement an implementation in Python.
The ultimate goal is that this implementation would be able to run arbitrary Python code,
and provide a reference implementation. While, it is unlikely to displace CPython as the de facto reference implementation, it will hopefully prove to be useful.
