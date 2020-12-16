# Top level semantics

The operational semantics consists of an abstract machine state, a start state,
a halt state and and a transition function.

The abstract machine, described more fully [here](./machine.md)
consists of a single interpreter.
We expect to change the model to include several interpreters as CPython supports that model of execution.

Each interpreter consists of a set of threads and a set of runnable threads

`runnable-threads ⊆ threads`

## The interpreter semantics

### Start state

The start state of an interpreter is as follows:

* There is exactly one thread and it is runnable.
* The code for that thread is the code for the interpreter.
* That thread is in its start state.

### Halt state

The halt state occurs when there are no runnable threads. 
That is, `runnable-threads = ∅`.

If there are no threads, `threads = ∅`, then the interpreter has terminated.
If there are threads, but none are runnable, then the interpreter has deadlocked.

### Execution steps

Execution of the abstract machine occurs in a, possibly infinite, number of steps.
Each step causes the abstract machine to transition from one state to another.
Each step is defined by a transition function and is an atomic, indivisible operation.

#### Transition function

The transition function for the interpreter is:

* Choose fairly a thread from `runnable-threads`
* Execute the transition function for that thread.

Here "fairly" is defined to mean that it tends to random when averaged over a large number of steps.

## Threads

The interpreter consists of a number of threads. Threads run concurrently,
but only one step is executed at a time per interpreter.

### Start state

To create the start state for a thread for a `code` object `c`:

* Start with an empty frame stack.
* Push a "halt frame", that is a frame with no locals, globals, or builtins, whose code object contains the single instruction `halt`, and `last` is set to -1.
* Push the frame for the `code` object `c`, as if `eval` was being called.

### Halt state

Threads do not have an explicit `halt` state, but when execution encounters a `halt` instruction, the thread is removed from the interpreter's `thread` set, and can thus never be run again.

### Transition function

The transition function of a thread is the core of the semantics.
The transition function is similar to the execution of real hardware in its most general form: fetch, then execute.

Real hardware has a decode phase, but an abstract machine has no encoding, so there is no need to decode.

The fetch phase simply sets the `last` value in the topmost frame to `next`, reads the instruction from `top_frame.code.instructions[next]`, then increments `next` by one.
The execute phase is described in [operations.md](./operations.md).



