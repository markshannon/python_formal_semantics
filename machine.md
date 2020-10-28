# Abstract machine description

The following describes the components of the abstract machine, and how they are related.

See [operations](./operations.md) for a description of the set of operation that operation on the abstract machine state.
See [translation](./translation.md) for a description of how Python source is translated into those operations.
See [objects](./objects.md) for a description of the core objects and classes necessary to implement Python.

The following model assumes a single intepreter, it should be easily extended to incorporate several communicating interpreters.
The current CPython implementation is a hybrid of these two models, but is attempting to move to the communicating interpreters model.

## Components

### Interpreter

The interpreter has the following attributes:

* Lock (the GIL)
* Thread list (zero or more threads)
* Reference to the currently running thread (if any)
* Heap -- Where objects are allocated. Once objects are no longer reachable, they will be automatically reclaimed.

### Thread

Each thread has the following attributes:

* `instruction pointer` Pointer to next instruction to be executed
* Frame stack

### Frame

Each frame has the following attributes:

* Data `stack`: used for computation
* `locals`: the local variables
* Reference to global variable mapping
* Reference to builtins dictionary
* `code`: a reference to the code object
* `state`: one of `CREATED`, `EXECUTING`, `SUSPENDED`, or `EXHAUSTED`
* `last`: index of last instruction executed in this frame
* `exit`: index of instruction to execute when exiting from callee generator

### Instructions

Execution in the abstract machine procedes by performing the operation described by instructions.
Each instruction has the following attributes:

* Name: name of the operation
* Operand: a name or number that specifies the exact behaviour of the operation.
* Line number: The line number of the source code coresponding to this instruction.

### Code

Each code object has the following attributes:

* List of arguments
* List of instructions.
* Source filename
* Names of all local variable names: For debugging purposes.

## Execution

Each thread in the machine repeated executes the current operation, until it halts.
In Python pseudo-code, this would be something like:
```python
while True:
    name, operand, line = get_current_instruction()
    if name == "halt":
        break
    execute_operation(name, operand)
```

### Running Python code

To run some Python code, either as a module or just a piece of source code,
the following steps must occur.

* Create an interpreter
* The Python code should be compiled to a code object for that interpreter.
* Create a thread within that interpreter
* Start the execution loop

### Running a code object in a thread

To run a code object in a thread, the thread's frame stack must be set up, then execution procedes as normal.
The frame stack is set up as follows:

* An entry frame is pushed to the thread's frame stack.
* A frame for the code object is created, and pushed to the thread's frame stack.

The entry frame is a minimal (no locals, globals, or builtins) frame, whose code object contains the one instruction: `halt`.
The entry frame's `last` is set to -1.

Normal execution will execute the code object. Once that returns, the `halt` instruction will be reached and execution will stop.
