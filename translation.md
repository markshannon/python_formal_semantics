# Translation from source code to a sequence of operations

In order to translate source code to a sequence of operations, it must first be parsed.
Parsing produces a tree representing the source code.
Correct parsing of the source code is complex, and out of scope for this documentation.

For this document we assume that source has been parsed to form a tree and that each syntactic part of the original source is represented by a subtree of the tree for the whole source.

We express the tranlsation from source to sequence of operations as a function on part of the AST than produces a sequence of operations. Translation is a recursive process, translation of the AST for `a + b` requires the result of translating `a` and of translating `b`. 

For convenience, the expression `translate(x)` means translate the AST for `x`.

### Terminology

#### Pseudo operations

In the following we will often describe the way one syntactic construct is translated by converting it to another one.
However, it is not always possible to do directly in Python source and we need to include a pseudo-operation.
We use a `!` to signify that a name is an pseudo-operation, not a Python variable.
For example, `load_attr` is just the Python variable "load_attr", but `load_attr!` means the "load_attr" pseudo-operation.

When describing translations and pseudo-operations, the pseudo-operations may used in what looks a normal Python call.
These should not be considered calls in Python, but rather like calls in a language like C, where there is no lookup and
the execution enters the pseudo-operation directly.

#### emit

The function `emit` emits the instruction to the imaginary buffer we are using to compile the code.

```python
def emit(opcode, operand=None):
    ...
```

## Expressions

### Attibutes

#### Load

The expression `a.b` is translated with help from the pseudo-operation `load_special` and
a call to the object's class's `__getattribute__` method.

`a.b` is translated by:

```python
    translate(a)
    emit(load_attr!, "b")
```

where `load_attr!` is defined as:

```python
def load_attr!(obj, name):
    _, getattr = load_special!(obj, "__getattribute__")
    assert(_ is True) # __getattribute__ is defined on object.
    return getattr(name)
```

## Load_special

Loading a special attribute from an object's class is a very common operation,
used to perform almost any operation.

```python
def load_special!(obj, name):
    cls = TYPE(obj)
    has_attr, descriptor = lookup_class_attr!(cls, name)
    if has_attr:
        desc_type = TYPE(descriptor)
        has_getter, getter = lookup_class_attr!(desc_type, '__get__')
        if has_getter:
            return True, getter(obj, cls)
        else:
            return True, descriptor
    return False, None
```

### lookup_class_attr!

```python
def lookup_class_attr!(cls, name):
    for scls in GET_MRO(cls):
        succ, result = LOOKUP_DICT(GET_DICT(scls), name)
        if succ:
            return result
    return False, None
```

### Calls

The call `f(*args, **kwrgs)` is translated using the `call!` pseudo-operation:

```python
    translate(f)
    translate(args)
    translate(kwargs)
    emit(call!)
```

Most calls are not of the form, `f(*args, **kwrgs)`. To translate those calls,
all positional arguments must be merged to form a tuple, and all named arguments must be
merged to form a dictionary.

Translation then procedes as follows:

1. Create an empty list
2. For each positional or star argument:
    * Tranlsate the argument
    * If star argument:
        * Extend the list
    * Else:
        * Emit `LIST_APPEND` operation.
3. Emit `LIST_TO_TUPLE`
4. Create an empty dict
5. For each named or double-star argument:
    * If double-star argument:
        * Merge into the dict, with `MERGE_DICT_NO_DUPLICATES`.
    * Else:
        * Tranlsate the key
        * Tranlsate the value
        * Emit `DICT_INSERT_NO_DUPLICATES` operation
6. Emit the `CALL` operation.


#### call!

The `call!` pseudo-operation is defined as follows:

```python
def call!(func, args, kwargs):
    while True:
        if TYPE(func) is types.FunctionType:
            frame = MAKE_FRAME(func, args, kwargs)
            ENTER_FRAME(frame)
            break
        if TYPE(func) is types.BuiltinFunctionType:
            if kwargs:
                raise TypeError(...)
            success, value = FFI_CALL(func, args)
            if success:
                PUSH(value)
            else:
                raise value
            break
        if TYPE(func) is types.MethodType:
            args = (func.self,) + args
            func = func.function
            continue
        is_callable, func = load_special!(func, "__call__")
        if not is_callable:
            raise TypeError("Not callable")
```

#### Example
    
The translation of `f(a, *b, c, x=1, **y, z=2)` is:

```
    LOAD_FAST 'f'
    BUILD_LIST 0
    LOAD_FAST 'a'
    LIST_APPEND 1
    LOAD_FAST 'b'
    LIST_EXTEND
    LOAD_FAST 'c'
    LIST_APPEND 1
    LIST_TO_TUPLE
    NEW_DICT
    LOAD_CONSTANT "x"
    LOAD_CONSTANT 1
    DICT_INSERT_NO_DUPLICATE
    LOAD_FAST 'y'
    MERGE_DICT_NO_DUPLICATES
    LOAD_CONSTANT "z"
    LOAD_CONSTANT 2
    DICT_INSERT_NO_DUPLICATE
    CALL
```

### Binary operations

Binary operations of the form `l op r` are all translated the same way, using the `binary_op` pseudo-operation.

`l op r` is translated by:

```python
    translate(l)
    tranlsate(r)
    opname, oprname = binary_op_names[op]
    emit(binary_op!, (opname, oprname))
```

where `binary_op_names` is a table defining the opnames.
Most opnames take the form `__xxx__`, `__rxxx__` where xxx is the mnemonic form of the operator.
For example, the opnames for `+` are `__add__` and `__radd__`.

#### binary_op!

`binary_op!` is defined as follows:

```
def binary_op!(l, r, name, rname):
    if SUBTYPE(TYPE(r), TYPE(l)) and r is not l: # Strict subclass
        r, l = l, r
        name, rname = rname, name
    succ, op = load_special!(l, name)
    if succ:
        res = op(r)
        if res is not NotImplemented:
            return res
    succ, op = load_special!(r, rname)
    if succ:
        res = op(l)
        if res is not NotImplemented:
            return res
    raise TypeError(...)
```

## Statements

### Assignments

#### Simple assignment

The translation of the simple assignment `a = b` depends on the scope in which `a` resides.
In all case the value `b` is translated first.

* If `a` is a function local variable then emit `store_local("a")`
* If `a` is a class local variable, then emit `store_name("a")`
* If `a` is a global variable, then emit `store_global("a")`
* If `a` is a non-local, that is a variable in an outer function, emit `store_nonlocal("a", n)` where `n` is the nesting difference between scope of the store and the scope of the variable.

Determining the scope of a variable requires symbol table analysis.

#### Attribute assignments

The statement `a.b = x` is translated with help from the operation `load_special` and
a call to the object's class's `__setattr__` method.

```
    translate(x)
    translate(a)
    emit(store_attr!, "b")
    POP() # discard result of call to store_attr
```

where `store_attr!` is defined as:

```python
def store_attr!(value, obj, name):
    _, setattrfunc = load_special!(obj, "__setattr__")
    assert _ is True # object has __setattr__, so it is always defined
    setattrfunc(name, value)
```

#### Indexed assignments

The statement `a[i] = x` is translated with help from the operation `load_special` and
a call to the object's class's `__setitem__` method.

```
    translate(x)
    translate(a)
    translate(i)
    emit(store_subscr!)
    POP() # discard result of call to store_subscr
```

where `store_subscr!` is defined as 

```python
def store_subscr!(value, obj, key):
    succ, func = load_special!(obj, "__setitem__")
    if succ:
        func(key, value)
    else:
        raise TypeError(...)
```

### Augmented assignments

Augmented assignments, such as `a += b` are implemented as if the operator and assignment were seperated, like `a = a + b`, but the any subexpression in `a` are only evaluated once and the inplace form of the operator is used.

`a += b` is translated by:

```python
    translate(a)
    translate(b)
    emit(inplace_op!, ("__iadd__", "__add__", "__radd__"))
    translate_store(a)
```

but `a.x += b` would be translated as:
```python
    translate(a)
    emit(STORE_TEMP, 'obj')
    emit(LOAD_TEMP, 'obj')
    emit(LOAD_ATTR, 'x')
    translate(b)
    emit(inplace_op!, ("__iadd__", "__add__", "__radd__"))
    emit(LOAD_TEMP, 'obj')
    emit(STORE_ATTR, 'x')
```

and `a[i] += b` is translated by:

```python
    translate(a)
    emit(STORE_TEMP, 'obj')
    emit(LOAD_TEMP, 'obj')
    translate(i)
    emit(STORE_TEMP, 'index')
    emit(LOAD_TEMP, 'index')
    emit(LOAD_SUBSCR)
    translate(b)
    emit(inplace_op!, ("__iadd__", "__add__", "__radd__"))
    emit(LOAD_TEMP, 'obj')
    emit(LOAD_TEMP, 'index')
    emit(STORE_SUBSCR)
```

where `inplace_op!` is defined as:

```python
def inplace_op!(l, r, iname, lname, rname):

    succ, op = load_special!(l, iname)
    if succ:
        res = op(r)
        if res is not NotImplemented:
            return res
    return binary_op!(l, r, lname, rname)
```

### Compound Statements and control flow


#### The `if` statement

```
if test:
    body
```

Is translated as:

```
    translate(test)
    emit(BRANCH, (False, end))
    translate(body)
end:
```

```
if test:
    body
else:
    elsebody
```

Is translated as:

```
    translate(test)
    emit(BRANCH, (False, orelse))
    translate(body)
    emit(JUMP, end)
orelse:
    translate(elsebody)
end:
```


#### Control flow stack

In order to correctly translate loops, `try` statements, `with` statements and anything else with complex control flow, we need to maintain a control flow stack. This is a stack of control flow structures that we are translating. It has no operational equivalent.
We define an anciliary function : `push_control(kind, loop_label, exit_label, ast)`.
This function is used in a with statement to temporarily push a control to the control stack.

#### The `while` statement

```
while test:
    body
```

Is translated by:

```
    tranlsate(test)
    emit(BRANCH, (False, end))
loop:
    with push_control("loop", (loop, end)):
        translate(body)
    emit(JUMP, loop)
end:
```

#### The `for` statement

```
for var in seq:
    body
```

Is translated as:

```
    translate(seq)
    emit(GET_ITER)
    emit(STORE_TEMP, "iter")
loop:
    emit(LOAD_TEMP, "iter")
    emit(FOR_ITER, end)
    translate_store(var)
    with push_control("loop", (loop, end)):
        translate(body)
    emit(JUMP, loop)
end:
```

#### The `try` `except` statement

```
try:
    body
except:
    handler
```

Is translated as:

```
    emit(PUSH_HANDLER(label)
    with push_control("try", None):
        translate(body)
    emit(POP_HANDLER)
    emit(JUMP, end)
label:
    emit(SWAP_EXCEPTION)
    emit(PUSH_HANDLER, cleanup)
    with push_control("handler", None):
        translate(handler)
    emit(POP_HANDLER)
    emit(SWAP_EXCEPTION)
    emit(POP_TOP)
    emit(JUMP, end)
cleanup:
    emit(ROT_TWO)
    emit(SWAP_EXCEPTION)
    emit(POP_TOP)
    emit(RERAISE)
end:
```

```
try:
    body
except ex as name:
    handler
```

is converted to:

```
try:
    body
except ex as name:
    handler
except:
    reraise!
```

which is translated as:

```
    emit(PUSH_HANDLER, label)
    with push_control("try", None):
        translate(body)
    emit(POP_HANDLER)
    emit(JUMP, end)
label:
    emit(DUP_TOP)
    emit(SWAP_EXCEPTION)
    emit(ROT_TWO)
    translate(ex)
    emit(EXCEPTION_MATCHES)
    emit(BRANCH_ON_FALSE, no_match)
    emit(PUSH_HANDLER, cleanup)
    translate_store(name)
    with push_control("handler", name):
        translate(handler)
    pop_handler!
    clear_local!(name)
    emit(JUMP, end)
cleanup:
    emit(ROT_TWO)
    emit(SWAP_EXCEPTION)
    translate_clear(name)
    emit(POP_TOP)
    emit(RAISE)
no_match:
    emit(RAISE)
end:
```

#### The `try` `finally` statement

```
try:
    body
finally:
    final_body
```

is translated as:

```
    emit(PUSH_HANDLER, finally_label)
    with push_control("finally", final_body):
        translate(body)
    emit(POP_HANDLER)
    translate(final_body)
    emit(JUMP, end)
finally_label:
    translate(final_body)
end:
```

#### The `with` statement

As described in PEP 343

```
with cm:
    body
```

Is translated as if the code were:

```
$exit_tmp = load_special!(cm, "__exit__")
load_special!(cm, "__enter__")()
try:
    body
except:
    if not $exit_tmp(*sys.exc_info()):
        raise
else:
    $exit_tmp(None, None, None)
```

```
with cm as var:
    body
```

Is translated as if the code were:

```
$exit_tmp = load_special!(cm, "__exit__")
$enter_tmp = load_special!(cm, "__enter__")()
try:
    var = $enter_tmp
    body
except:
    if not $exit_tmp(*sys.exc_info()):
        raise
else:
    $exit_tmp(None, None, None)
```

### Continue, break and return statements

The `continue`, `break` and `return` statements transfer control, but care must to taken
to ensure that `with` and `finally` statements are handled correctly.
To do so, the control stack must be traversed and the relevant code emitted.
For that we define the helper function, `emit_control(name, data)` which emits the
code to cleanup the control statement.

NOTE:
* TO DO -- This needs to be double checked, the "handler" code is probably wrong.

```python
def emit_control(name, data):
    if name == "loop":
        pass
    elif name == "try":
        emit(POP_HANDLER)
    elif name == "finally":
        emit(POP_HANDLER)
        translate(data)
    elif name == "finally_cleanup":
        emit(POP)
        emit(POP_HANDLER)
        emit(SET_EXCEPTION)
    elif name == "handler":
        emit(POP_HANDLER)
        if data is not None:
            emit(POP_HANDLER)
        emit(POP_EXCEPT)
        if data is not None:
            emit(CLEAR_LOCAL, data)
```

### The `break` statement

For a `break` statement, the control stack is unwound, until a "loop" control is found.
Then a `jump` is to the exit is emitted.

```python
for name, data in control_stack:
    emit_control(name, data)
    if name == "loop":
        loop, exit = data
        emit(JUMP, exit)
        break
else:
    raise SyntaxError("break not in loop")
```

### The `continue` statement

The continue is much like the `break` statement, except that it jumps to the loop, not the exit.


```python
for name, data in control_stack:
    emit_control(name, data)
    if name == "loop":
        loop, exit = data
        emit(JUMP, loop)
        break
else:
    raise SyntaxError("continue not in loop")
```

### The `return` statement


```python
# Save return value before unwinding
emit(STORE_TEMP, "return_value")
for name, data in control_stack:
    emit_control(name, data)
emit(LOAD_TEMP, "return_value")
emit(RETURN)
```