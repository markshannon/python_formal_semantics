# Operations supported by the abstract machine

## Call to builtin functions

Calls to builtin functions are performed by the foreign function interface call operation, `fficall`
For the purposes of this sematics, builtin functions can take only a fixed number of positional arguments.
More complex interfaces are possible by wrapping the builtin functions in a Python function.

### fficall

The `fficall` operation has one integer operand, `argcount`.
It pops argcount values off the stack, pops the callable, and passes the arguments to the builtin function.
If the call is successful, the result is pushed to the stack.
If the call fails, then unwinding occurs.

## Calls to Python functions.

Calls to Python functions are performed in a two step process. Frame creation and pushing the frame.

### make_frame

This operation has no operands.
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

## Calls to bound methods

Calls to bound methods are performed by prepending the `self` attribute of the bound method to the tuple,
replacing the callable with the `function` attribute of the bound method and repeating the call procedure.

## General calls

If the callable is a Python function, a builtin function, or a bound method,
then procede as above. Otherwise replace the callable with the result of evaluating `load_special("__call__")` on the callable,
then repeat the call procedure:

```python
call():
    kwargs = POP()
    args = POP()
    func = POP()
    while True:
        if type!(func) is types.FunctionType:
            frame = make_frame!(args, kwargs)
            enter_frame!(frame)
            break
        if type!(func) is types.BuiltinFunctionType:
            if kwargs:
                unwind!(error!(TypeError, ...))
            success, value = fficall!(func, args)
            if success:
                PUSH(value)
            else:
                unwind!(value)
            break
        if type!(func) is types.MethodType:
            args = (func.self,) + args
            func = func.function
            continue
        func = load_special!(func, "__call__")
```


### enter_frame 

This operation has no operands.
Sets `last` to the index of the instruction pointed to by the thread's instruction pointer.
Pushes the frame to the top of the current thread's frame stack.
Sets the thread's instruction pointer to point to the frame's `last` instruction plus one.

### return

This operation has no operands.
Pops the value from the stack. Note: this should be the only value on the stack.
Pops the frame from the thread's frame stack.
Pushes the value to the stack.

## Load_special

Loading a special attribute from an object's class is a common operation.

```python
load_special(name):
    obj = POP()
    cls = type!(obj)
    if has_class_attr!(cls, name): # Has class attr
        descriptor = get_class_attr!(cls, name) # Get class attr
        desc_type = type!(descriptor)
        if has_class_attr!(desc_type, '__get__'):
            getter = get_class_attr!(desc_type, '__get__')
            PUSH(descriptor)
            PUSH((obj, cls))
            PUSH({})
            call!()
        else:
            PUSH(descriptor)
    else:
        msg = ... # "'{cls.__name__}' object has no attribute '{name}'"
        unwind!(AttributeError(msg))
```

## Binary operations

```python
binary_op(lname, rname):
    right = POP()
    left = POP()
    if subtype!(type!(right), type!(left)):
        PUSH(right)
        func = load_special!(rname)
        PUSH(left)
        PUSH(right)
        call!()
    else:
        PUSH(left)
        func = load_special!(lname)
        PUSH(left)
        PUSH(right)
        call!()
        result = POP()
        if result is NotImplemented:
            PUSH(right)
            func = load_special!(rname)
            PUSH(left)
            PUSH(right)
            call!()
    result = POP()
    if result is NotImplemented:
        unwind!(TypeError("operator not implemented"))
    PUSH(result)
```

`a + b` is implemented as:
```
    #compile a
    #compile b
    binary_op("__add__", "__radd__")
```

The other operators follow the same pattern.

## Halt

A special operation, `halt`, exists for terminating execution of a thread.
All other operations implicitly procede to executing the next operation.
`halt` does not; execution of the thread halts.

## Control Flow

Three are three control flow operations:

* ``jump`` -- Jumps uncondtionally to the target label.
* ``branch_on_true`` -- Pops the top value on the stack and jumps to the target label, if true.
* ``branch_on_false`` -- Pops the top value on the stack and jumps to the target label, if false.