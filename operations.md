# Operations supported by the abstract machine

This file lists the operations on the abstract machine.

## Calls and helpers


### FFI_CALL

This opcode makes calls to builtin functions. The operand is the number of arguments

```
builtin_func *args -> result
```

Pops the arguments and builtin-function and makes the call using the foreign function interface.
If the call is successful, the result is pushed to the stack.
If the call fails, then unwinding occurs.

For the purposes of this semantics, builtin functions can take only a fixed number of positional arguments.
More complex interfaces are possible by wrapping the builtin function in a Python function.

### Calls to Python functions.

Calls to Python functions are performed in a two step process. Frame creation and pushing the frame.

### MAKE_FRAME

This operation has no operand.

```
func args_tuple keyword_dict -> frame
```

The function, a tuple of positional arguments, and a dictionary of keyword arguments are popped from the stack.
The dictionary is top of the stack, next the positional arguments, then the function.
The frame is initialized as follows:

* If the length of the tuple is greater than the number of arguments, excluding keyword-only arguments, then fail.
* Move the values in the tuple into the locals, starting at position 0.
* For each key, value pair in the keyword arguments:
    * if the key is not a string, then fail.
    * if there is no argument of that name, then fail.
    * if that argument has already been assigned, then fail.
    * if that argument is positional-only, then fail.
    * move the value to the named argument
* For each argument so far unassigned:
    * Assign the default value, if there is one. Otherwise, fail.
* Copy the `globals`, `builtins`, and `code` attributes from the function.
* Set `state` to `CREATED`
* Set `last` to -1

If frame creation fails, then perform unwinding with a `TypeError`.

### ENTER_FRAME

```
frame -> 
```

This operation has no operands.
Sets `last` attribute of the frame currently on top of the stack to the thread's `next` attribute.
Pushes the frame to the top of the current thread's frame stack.
Sets the thread's `next` attribute to the frame's `last` attribute, plus one.

### RETURN

This operation has no operands.

Callee:
```
    value ->
```

then caller:
```
    -> value
```

Pops the value from the data stack. Note: this should be the only value on the stack.
Pops the frame from the thread's frame stack.
Pushes the value to the data stack.

### Call Helpers

To implement calls, we need a few helper operations to build up the arguments.

### LIST_APPEND

```
list item -> list
```

Append `item` to `list`.

### DICT_INSERT_NO_DUPLICATE

```
dict key value -> dict
```

Insert `key, value` pair into `dict`, raising an Exception if `key` is already present in `dict`.

### LIST_TO_TUPLE

```
list -> tuple
```

Convert `list` to `tuple`.

### MAPPING_TO_DICT

```
obj -> dict
```

Convert `obj` to a mapping. Fail if `obj` is not a mapping.

## Type checking operations

### TYPE

Takes no operand.

```
obj -> cls
```

Pops the object on top of the stack and pushes its class.

### SUBTYPE

Takes no operand.

```
cls1 cls2 -> res
```

Pushes `True` if `cls1` is a direct subtype of `cls2`,
ignoring anything but the class's MRO.
```
res = cls1 in cls2.__mro__
```

## Control Flow

Three are three local control flow operations:

* ``JUMP`` -- Jumps unconditionally to the target label.
* ``BRANCH`` -- Pops the top value on the stack and jumps to the target label, if value matches condition.

And non-local control flow operations
* ``RETURN`` -- Returns from a call, see section on calls.
* ``RAISE`` -- Starts unwinding, see section on exceptions.
* ``HALT`` -- Stops the current thread.

In addition there is a `TO_BOOL` operation to convert any value to a boolean.
This is provided to avoid have to mix the conversion to boolean and the jump into a single operation.

### JUMP

```
->
```

This operation has one operand, the `offset`, as a signed integer, of the target instruction from the following instruction.

The `jump` instruction simply adds the `offset` to the `next` attribute of the current thread.

### BRANCH

```
cond ->
```

This operation has two operands, the `way` (a boolean), and the `offset`, as a signed integer, of the target instruction from the following instruction.

The `BRANCH` operation pops the value from the top of the stack and, if the value is the same as `way`, adds the `offset` to the `next` attribute of the current thread.

```python
def BRANCH(cond, way, target):
    if cond is way:
        JUMP(target)
```

### TO_BOOL

```
obj -> bool
```

This operation has no operands.
Converts the obj on top of the stack to a bool. 
This operation is not strictly necessary as it is defined as:

```python
def TO_BOOL(obj):
    has_bool, to_bool = load_special!(obj, "__bool__")
    if has_bool:
        res = to_bool()
        if res is True or res is False:
            return res
        raise TypeError(...)
    return True
```

### HALT

Halt has no operands.

A special operation, `HALT`, exists for terminating execution of a thread.
All other operations implicitly proceed to executing the next operation.
`HALT` does not; execution of the thread halts.

`HALT` removes the current thread from the interpreter's `threads` and `runnable-threads` sets.


## Exception Handling

The are three exception handling control operations:

### PUSH_HANDLER

```
->
```

Pushes an exception handler to the handler stack.
Takes one operand, the offset, as a signed integer, of the target instruction from the following instruction.

This instruction pushes the pair `(stackdepth, target)` to the handler stack,
where `stackdepth` is the current depth of the data stack.

#### POP_HANDLER

Has no operand.

```
->
```

Pops the top pair from the handler stack.

#### SWAP_EXCEPTION

Swaps the exception on top of the data stack with the current exception

```
exc -> current_exc
```
Sets `current_exc = exc`.
