# Translation from source code to a sequence of operations

In order to translate source code to a sequence of operations, it must first be parsed.
Parsing produces a tree representing the source code.
Correct parsing of the source code is complex, and out of scope for this documentation.

For this document we assume that source has been parsed to form a tree and that each syntactic part of the original source is represented by a subtree of the tree for the whole source.

We express the tranlsation from source to sequence of operations as a function on part of the AST than produces a sequence of operations. Translation is a recursive process, translation of the AST for `a + b` requires the result of translating `a` and of translating `b`. 

For convenience, the expression `translate(x)` means translate the AST for `x`.

## Expressions

### Attibutes

#### Load

The expression `a.b` is translated with help from the operation `load_special` and
a call to the object's class's `__getattribute__` method.

`a.b` is translated by:

```python
    translate(a)
    emit(STORE_TEMP('obj'))
    emit(LOAD_TEMP('obj'))
    emit(LOAD_SPECIAL("__getattribute__"))
    emit(LOAD_TEMP('obj'))
    emit(BUILD_TUPLE(1)) # (obj,)
    emit(NEW_DICT) # No keyword arguments
    emit(CALL)
```

### Calls

The call `f(*args, **kwrgs)` is translated using the `call` operation:

```python
    translate(f)
    translate(args)
    translate(kwargs)
    emit(CALL)
```

Most calls are not of the form, `f(*args, **kwrgs)`. To translate those calls,
all positional arguments must be merged to form a tuple, and all named arguments must be
merged to form a dictionary.

To do this we need to the following helper operations:
* `list_append(list, item)` - Append `item` to `list`, leaving `list` on the stack.
* `dict_insert_no_duplicate` - Pop `value`, `key` and `dict` from stack, in that order. Insert `key, value` pair into `dict`, raising an Exception if `key` is already present in `dict`. Push `dict` back on the stack.
* `list_to_tuple(list)` - Pops the `list` on top of the stack and pushes a tuple with the same values.
* `mapping_to_dict(obj)` - Pops the `obj` on top of the stack, failing if not a mapping, and pushes a dict to the stack with the same values.


Translation then procedes as follows:

1. Create an empty list
2. For each positional or star argument:
    * Tranlsate the argument
    * If star argument:
        * Extend the list
    * Else:
        * Emit `list_append` operation.
3. Emit `list_to_tuple`
4. Create an empty dict
5. For each named or double-star argument:
    * If double-star argument:
        * Merge into the dict, with `merge_dict_no_duplicates`.
    * Else:
        Tranlsate the key
        Tranlsate the value
        * Emit `dict_insert_no_duplicate` operation
6. Emit the `call` operation.



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

Binary operations of the form `l op r` are all translated the same way, using the `binary_op` instruction.

`l op r` is translated by:

```python
    translate(l)
    tranlsate(r)
    opname, oprname = binary_op_names[op]
    emit(BINARY_OP(opname, oprname))
```

where `binary_op_names` is a table defining the opnames.
Most opnames take the form `__xxx__`, `__rxxx__` where xxx is the mnemonic form of the operator.
For example, the opnames for `+` are `__add__` and `__radd__`.

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
    obj = POP()
    value = POP()
    setattrfunc = load_special!(obj, "__setattr__")
    PUSH(setattrfunc)
    PUSH((obj, value))
    PUSH({})
    call!
    POP() # discard result of call.
```

#### Indexed assignments

The statement `a[i] = x` is translated with help from the operation `load_special` and
a call to the object's class's `__setitem__` method.

```
    translate(x)
    translate(a)
    translate(i)
    index = POP()
    obj = POP()
    value = POP()
    setitemfunc = load_special!(obj, "__setitem__")
    PUSH(setitemfunc)
    PUSH((obj, index, value))
    PUSH({})
    call!
    POP() # discard result of call.
```

### Augmented assignments

Augmented assignments, such as `a += b` are implemented as if the operator and assignment were seperated, like `a = a + b`, but the any subexpression in `a` are only evaluated once and the inplace form of the operator is used.

`a += b` is translated by:

```python
    translate(a)
    translate(b)
    emit(INPLACE_OP("__iadd__"))
    translate_store(a)
```

but `a.x += b` would be translated as:
```
    translate(a)
    emit(STORE_TEMP('obj'))
    emit(LOAD_TEMP('obj'))
    emit(LOAD_ATTR('x'))
    translate(b)
    emit(INPLACE_OP("__iadd__"))
    emit(LOAD_TEMP('obj'))
    emit(STORE_ATTR('x'))
```

and `a[i] += b` is translated by:

```
    translate(a)
    emit(STORE_TEMP('obj'))
    emit(LOAD_TEMP('obj'))
    translate(i)
    emit(STORE_TEMP('index'))
    emit(LOAD_TEMP('index'))
    emit(LOAD_SUBSCR)
    translate(b)
    emit(INPLACE_OP("__iadd__"))
    emit(LOAD_TEMP('obj'))
    emit(LOAD_TEMP('index'))
    emit(STORE_SUBSCR)
```

### Compound Statements and control flow


#### The `if` statement

```
if test:
    body
```

Is translated as:

```
    tranlsate(test)
    branch_on_false!(end)
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
    tranlsate(test)
    branch_on_false!(orelse)
    translate(body)
    jump!(end)
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
    emit(BRANCH_ON_FALSE(end))
loop:
    with push_control("loop", (loop, end)):
        translate(body)
    emit(JUMP(loop))
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
    emit(STORE_TEMP("iter"))
loop:
    emit(LOAD_TEMP("iter"))
    emit(FOR_ITER(end))
    translate_store(var)
    with push_control("loop", (loop, end)):
        translate(body)
    emit(JUMP(loop))
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
    push_handler!(label)
    with push_control("try", None):
        translate(body)
    pop_handler!()
    jump!(end)
label:
    translate(handler)
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
    push_handler!(label)
    with push_control("try", None):
        translate(body)
    pop_handler!
    jump!(end)
label:
    dup_top!
    translate(ex)
    exception_matches!
    branch_on_false!(no_match)
    push_handler!(cleanup)
    with push_control("handler", name):
        translate(handler)
    pop_handler!
    clear_local!(name)
    jump!(end)
no_match:
    reraise!
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
    push_handler!(finally_label)
    with push_control("finally", final_body):
        translate(body)
    pop_handler!
    translate(final_body)
    jump!(end)
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
normal_exit = True
try:
    body
except:
    normal_exit = False
    if not $exit_tmp(*sys.exc_info()):
        raise
finally:
    if normal_exit:
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
normal_exit = True
try:
    var = $enter_tmp
    body
except:
    normal_exit = False
    if not $exit_tmp(*sys.exc_info()):
        raise
finally:
    if normal_exit:
        $exit_tmp(None, None, None)
```

### Continue, break and return statements

The `continue`, `break` and `return` statements transfer control, but care must to taken
to ensure that `with` and `finally` statements are handled correctly.
To do that, the control stack must be traversed and the relevant code emitted.
To do that we define the helper function, `emit_control(name, data)` which emits the
code to cleanup the control statement.

```python
def emit_control(name, data):
    if name == "loop":
        pass
    elif name == "try":
        emit(POP_HANDLER)
    elif name == "finally"
        emit(POP_HANDLER)
        translate(data)
    elif name == "finally_cleanup":
        emit(POP)
        emit(POP_EXCEPT)
    elif name == "handler":
        emit(POP_HANDLER)
        if data is not None:
            emit(POP_BLOCK)
        emit(POP_EXCEPT)
        if data is not None:
            emit(CLEAR_LOCAL(data))
```


### The `break` statement

For a `break` statement, the control stack is unwound, until a "loop" control is found.
Then a `jump` is to the exit is emitted.

```python
for name, data in control_stack:
    emit_control(name, data)
    if name == "loop":
        loop, exit = data
        emit(JUMP(exit))
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
        emit(JUMP(loop))
        break
else:
    raise SyntaxError("continue not in loop")
```