# Iteration and generators examples

## Introduction

This is *not* the final formal semantics of iteration and generators, but it is close enough to be illustrative.

See `Core.md` for details of the VM state, abbreviations and low-level instructions.

## Iteration

The loop

```python
    for it in seq:
        body
```

Will compile into the following instruction sequence:

```
    <Code to load 'seq'>
    GET_ITER
    FOR_ITER exit
    <Code to save TOS to 'it'>
    <Code to load 'it' to TOS>
    <body>
exit:
    POP_TOP # discard the return value from the generator
    POP_TOP # discard the generator
```

### Explanation of instructions

#### GET_ITER

The `GET_ITER` instruction replaces TOS with a generator suitable for iterating over that value.
If TOS is a generator then this is a no op, otherwise:

1. Replace TOS with result of the C-level call to the `tp_iter` slot of the type of TOS.
2. If TOS is not a generator, wrap it with the generator resulting from `wrap(TOS)`.
3. Increment the `instruction pointer`

`wrap` is defined as:

```python
def wrap(obj):
    try:
        yield $ITERNEXT(obj)
    except StopIteration:
        return
```

`$ITERNEXT(obj)` is not a call, and compiles to:

```
    <Code to load 'obj'>
    ITERNEXT
```

#### ITERNEXT

The `ITERNEXT` instruction makes a C-level call to the `tp_iternext` slot of the type of TOS. 
If NULL is returned, but no exception set by the C code, it sets the exception as `StopIteration`.

#### FOR_ITER

The `FOR_ITER` instruction does the following:

1. Set the `last` attribute of the current frame to the current instruction.
2. Set the `exit` attribute of the current frame to the instruction following the `exit:` label.
3. Push the frame of the generator in TOS to the current thread's frame stack.
4. Push `None` to the current frame's data stack.
5. Set the frame's `state` to `EXECUTING`
6. Set the `instruction pointer` to the instruction after the `last` attribute of the newly pushed frame.
3. `Continue`

## Generators

The generator function

```python
def gen():
    yield 1
    return "done"
```

Will compile into the following instruction sequence:

```
    MAKE_GEN
    YIELD_VALUE
    POP_TOP
    LOAD_CONSTANT 1
    POP_TOP
    LOAD_CONSTANT "done"
    GEN_RETURN
exhausted:
    EXHAUSTED
    JUMP exhausted
```

### Explanation of instructions

#### MAKE_GEN

The `MAKE_GEN` creates a generator from the current frame add pushes it to the stack.

It performs the following steps:

1. Create a generator object who's frame is the current frame.
2. Push that generator to the stack.
3. Increment the `instruction pointer`
4. Continue

#### YIELD_VALUE

The `YIELD_VALUE` instruction does the following:

1. Store `instruction pointer` in the `last` attribute of the current frame.
2. Set the frame's `state` to `SUSPENDED`
3. Pop value from current frame’s data stack
4. Pop current frame from the thread's frame stack.
5. Push the popped value to current frame’s data stack.
6. Set the `instruction pointer` to the instruction after the `last` attribute of the newly pushed frame.
7. `Continue`

#### GEN_RETURN

The `GEN_RETURN` instruction does the following:

1. Store `instruction pointer` in the `last` attribute of the current frame.
2. Set the frame's `state` to `EXHAUSTED`
3. Pop value from current frame’s data stack
4. Pop current frame from the thread's frame stack.
5. Push the popped value to current frame’s data stack.
6. Set the `instruction pointer` to the instruction the `exit` attribute of the newly pushed frame.
7. `Continue`

#### EXHUASTED

The `EXHUASTED` instruction is equivalent to 
```
    LOAD_CONSTANT None
    GEN_RETURN
```
but does not use the stack, to allow implementations to free the memory for the stack as soon as the generate is exhausted.







