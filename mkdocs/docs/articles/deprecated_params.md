---
title: Deprecated Parameters
summary: How to deprecate function parameters
authors:
    - Laurent LAPORTE
date: 2021-11-03
---

Deprecated Parameters
=====================

Abstract
--------

The purpose of this article is to present the case where we want to
deprecate one (or more) function parameters. This case is slightly
different from the case where we wish to deprecate a function because it
is necessary to examine the parameters of the function, thus necessary
to make an introspection.

-   First, we will define the terms *parameter* and *argument*.
-   Then, we will give some simple examples of functions for which we
    want to deprecate one (or more) parameter(s).
-   We can see how to decorate these functions with a "deprecated
    parameters" decorator.
-   We will also see how to introspect a function in order to list the
    positional and keyword parameters, and how parameters are bind to
    arguments.
-   This analysis will allow us to design a simple decorator to
    deprecate parameters.

Parameters or Arguments?
------------------------

The terms *parameter* and *argument* can be used for the same thing:
information that are passed into a function.

From a function's perspective:

-   A parameter is the variable listed inside the parentheses in the
    function definition.
-   An argument is the value that are sent to the function when it is
    called.

Example:

```python
def add(x, y):
    return x + y

add(4, 8)
```

In this example, we define a function `add` which has 2 **parameters**:
`x` and `y`. We call this function with the **arguments** 4 and 8.

Some examples
-------------

Here are some examples of functions for which we want to deprecate one
(or more) parameter(s).

### Example 1

We have, for example, the case where a function has an optional
parameter that is not used. The useless parameter is a positional (or
keyword) parameter.

```python
def pow2(x, y, z=None):
    return x ** y
```

Python 3.8 and above support positional-only parameters. So, this
function can be rewritten as follows:

```python
def pow2(x, y, z=None, /):
    return x ** y
```

➢ Classic usage:

```python-repl
>>> pow2(2, 3)
8
>>> pow2(2, 3, 3.14)
8
>>> pow2(2, 3, z=3.14)
8
```

### Example 2

A function can have keyword-only parameters.

The example below is a comparison function that takes two parameters and
a keyword-only parameter. In this example, we will consider that the use
of the *key* parameter should be deprecated say for security reasons.

```python
def compare(a, b, *, key=None):
    if key is None:
        return a < b
    return key(a) < key(b)
```

➢ Classic usage:

```python-repl
>>> compare("a", "B")
False
>>> compare("a", "B", key=str.upper)
True
>>> compare([2, 1], [1, 2], key=lambda i: i.pop())
True
```

### Example 3

A function can contain variable positional and keywords parameters.
In this case, it will be possible to deprecate the
keyword parameters but not the positional parameters.

This example shows how to implement a function that accepts two
positional parameters or two keyword parameters (`x, y` or
`width, height`). A warning message should be emitted if `x` and `y` are
used instead of `width` and `height`.

```python
def area(*args, **kwargs):

    def _area_impl(width, height):
        return width * height

    if args:
        # positional arguments (no checking)
        return _area_impl(*args)
    elif set(kwargs) == {"width", "height"}:
        # nominal case: no warning
        return _area_impl(kwargs["width"], kwargs["height"])
    elif set(kwargs) == {"x", "y"}:
        # old case: deprecation warning
        return _area_impl(kwargs["x"], kwargs["y"])
    else:
        raise TypeError("invalid arguments")
```

➢ Classic usage:

```python-repl
>>> area(4, 6)
24
>>> area(width=3, height=6)
18
>>> area(x=2, y=7)
14
```

The "deprecated parameters" decorator
-------------------------------------

We will assume that we have a decorator named `@deprecated_params` that
allows us to deprecate function parameters. This decorator must allow to
list the deprecated parameters and to define warning messages which will
be emitted when the said parameters are used.

Classic usage with a default warning message:

```python
@deprecated_params("z")
def pow2(x, y, z=None):
    return x ** y
```

Another usage where we use a dictionary which keys are the parameter
names and the values are the warning messages:

```python
@deprecated_params({"key": "Parameter 'key' should be avoided for security reasons"})
def compare(a, b, *, key=None):
    ...
```

The usage of a dictionary allows to indicate that several parameters are
deprecated:

```python
@deprecated_params(
    {
        "x": "Use 'width' instead",
        "y": "Use 'height' instead",
    }
)
def area(*args, **kwargs):
    ...
```

> **note**
>
> Of course, a more elaborate version will have to provide the API
> version number from which the parameters are deprecated. It will also
> be necessary to be able to redefine the warning class that is used
> similarly to the `@deprecated` decorator.

Binding positional and keyword parameters
-----------------------------------------

The `inspect` module contains a function to read the signature of a
function (or method). The signature is an object that can be linked to
the arguments passed to the function.

**Case 1.** Suppose we want to inspect the signature of the function `pow2`. Depending on how we call the function, we
will see if the `z` parameter is used or not. Here is an example:

```python-repl
>>> sig = inspect.signature(pow2)
>>> sig.bind(2, 3)
<BoundArguments (x=2, y=3)>
>>> sig.bind(2, 3, 4)
<BoundArguments (x=2, y=3, z=4)>
```

In the last call, `z` appears in the bound arguments.

**Case 2.**  The `compare` function has a different signature, but the binding between parameters and arguments works
the same. The `bind` method creates a mapping from positional and keyword arguments to parameters. It raises
a `TypeError` if the passed arguments do not match the signature.

```python-repl
>>> sig = inspect.signature(compare)
>>> sig.bind("a", "B")
<BoundArguments (a='a', b='B')>
>>> sig.bind("a", "B", key=str.upper)
<BoundArguments (a='a', b='B', key=<method 'upper' of 'str' objects>)>
>>> sig.bind("a", "B", str.upper)
Traceback (most recent call last):
...
TypeError: too many positional arguments
```

**Case 3.**  In the third case with the `area` function, the binding is done with the positional and/or keyword
parameters.

```python-repl
>>> sig = inspect.signature(area)
>>> sig.bind(4, 6)
<BoundArguments (args=(4, 6))>
>>> sig.bind(x=4, y=6)
<BoundArguments (kwargs={'x': 4, 'y': 6})>
>>> sig.bind(width=4, height=6)
<BoundArguments (kwargs={'width': 4, 'height': 6})>
```

A simple decorator factory
--------------------------

The decorator we have to design has necessarily initialization
parameters: we have to indicate which parameters are deprecated and also
provide warning messages. We call this kind of decorator a decorator
factory: this is a function (or a callable class) which define a
decorator.

To implement this decorator, an initialisation phase, a setup phase and
a control phase must be defined.

-   The initialisation phase is used to check the decorator's arguments
    and to build warning messages in advance.
-   The setup phase allows to extract the signature of the function in
    one go.
-   The control phase allows the binding between the function parameters
    and the arguments. We can then control the parameters from this
    binding and emit warning messages when we find deprecated
    parameters.

Here is a first version of such a decorator:

```python
import collections
import functools
import inspect
import itertools
import warnings


class DeprecatedParamsV1(object):
    def __init__(self, param, reason="", category=DeprecationWarning):
        self.messages = collections.OrderedDict()
        self.category = category
        self.build_messages(param, reason=reason)

    def build_messages(self, param, reason=""):
        if isinstance(param, dict):
            self.messages.update(param)
        elif isinstance(param, str):
            reason = reason or "'{param}' parameter is deprecated".format(param=param)
            self.messages[param] = reason
        else:
            raise TypeError(param)

    def check_params(self, signature, *args, **kwargs):
        # fmt: off
        binding = signature.bind(*args, **kwargs)
        return [
            param
            for param in itertools.chain(binding.arguments, binding.kwargs)
            if param in self.messages
        ]
        # fmt: on

    def warn_messages(self, messages):
        for message in messages:
            warnings.warn(message, category=self.category, stacklevel=3)

    def __call__(self, f):
        # type: (callable) -> callable
        signature = inspect.signature(f)

        @functools.wraps(f)
        def wrapper(*args, **kwargs):
            invalid_params = self.check_params(signature, *args, **kwargs)
            self.warn_messages([self.messages[param] for param in invalid_params])
            return f(*args, **kwargs)

        return wrapper
```

> **warning**
>
> In the examples below, we have changed the warnings configuration to
> raise an exception when a deprecation message is emitted. This makes
> it easier to see the messages when this code is tested with `doctest`.
> Usually, warning messages appear on the console in `stderr`.

Usage examples:

```python
@DeprecatedParamsV1("z")
def pow2(x, y, z=None):
    return x ** y
```

When we call the function `pow2` in different ways, we have:

```python-repl
>>> pow2(2, 3)
8

>>> pow2(2, 3, 3.14)
Traceback (most recent call last):
  ...
DeprecationWarning: 'z' parameter is deprecated

>>> pow2(2, 3, z=3.14)
Traceback (most recent call last):
  ...
DeprecationWarning: 'z' parameter is deprecated
```

Here is another example with the `compare` function:

```python
@DeprecatedParamsV1({"key": "Parameter 'key' should be avoided for security reasons"})
def compare(a, b, *, key=None):
    if key is None:
        return a < b
    return key(a) < key(b)
```

When we call the function `compare`, we have:

```python-repl
>>> compare("a", "B")
False

>>> compare("a", "B", key=str.upper)
Traceback (most recent call last):
  ...
DeprecationWarning: Parameter 'key' should be avoided for security reasons

>>> compare([2, 1], [1, 2], key=lambda i: i.pop())
Traceback (most recent call last):
  ...
DeprecationWarning: Parameter 'key' should be avoided for security reasons
```

Again, we can use the decorator with the `area` function:

```python
@DeprecatedParamsV1(
    {
        "x": "Use 'width' instead",
        "y": "Use 'height' instead",
    }
)
def area(*args, **kwargs):

    def _area_impl(width, height):
        return width * height

    if args:
        # positional arguments (no checking)
        return _area_impl(*args)
    elif set(kwargs) == {"width", "height"}:
        # nominal case: no warning
        return _area_impl(kwargs["width"], kwargs["height"])
    elif set(kwargs) == {"x", "y"}:
        # old case: deprecation warning
        return _area_impl(kwargs["x"], kwargs["y"])
    else:
        raise TypeError("invalid arguments")
```

When we call the function `area`, we have:

```python-repl
>>> area(4, 6)
24

>>> area(width=3, height=6)
18

>>> area(x=2, y=7)
Traceback (most recent call last):
  ...
DeprecationWarning: Use 'width' instead
```
