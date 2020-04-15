# Core data structures forming the VM state

The following model assumes a single intepreter, it should be easily extended to incorporate several communicating interpreters.

The current CPython implementation is a hybrid of these two models, but is attempting to move to the communicating interpreters model.

## Interpreter

The interpreter has the following attributes:

* Lock (the GIL)
* Thread list
* Reference to the currently running thread (if any)

## Thread

Each thread has the following attributes:

* `instruction pointer` Pointer to next instruction to be executed
* Frame stack

## Frame

Each frame has the following attributes:

* Data `stack`: used for computation
* `locals`: the local variables
* Reference to global variable mapping
* Reference to builtins dictionary
* `code`: a reference to the code object
* `state`: one of `CREATED`, `EXECUTING`, `SUSPENDED`, or `EXHAUSTED`
* `last`: index or pointer to last instruction executed in this frame
* `exit`: index or pointer to instruction to execute when exiting from callee generator

## Code object

Each code has the  following attributes

* List of arguments
* List of instructions.
* Exception jump table. Where excution should jump to if an exception is raised.