Requirements
============

---

- title: Deprecate a function parameter – Requirements
- summary: What are the requirements to implement a parameter deprecation decorator?
- author: Laurent LAPORTE
- date: 2019-03-01

---

Abstract
--------

When a program evolves, we sometimes have to change a function signature.
But this leads to a version incompatibility which is not sustainable by the user.

Instead of doing this, it is often easier to slightly change the function parameters,
for instance by adding keyword parameters (or by using `*args`/`**kwargs`).
Then it is better to emit a warning when the user uses an obsolete parameter.

There are of course other use cases where we want to deprecate a parameter, for instance when a parameter is renamed:
we keep the old parameter and add a new one, both parameters becoming optional. Then we need to add a code snippet
to check the parameters to make sure that at least one of them is used. Then we emit a warning message when the old
parameter is used.

The purpose of this article is to define the requirements to implement a decorator. To do so, we will give the
advantages and disadvantages of a decorator. Then we will try to generalize the requirements to find out if other use
cases could not also be taken into account. We will continue by looking for examples of deprecated parameters in open
source libraries. Finally, we will discuss the implementation of such a decorator, taking inspiration from what is
already done in the Deprecated library.

To illustrate our point, here is an example of a function to create a styled paragraph in HTML. Here, the *color*
and *background_color* parameters are replaced by a more generic *style* parameter. Both parameters are now deprecated.

```python
import xml.sax.saxutils


@deprecated_params(
    params='color background_color',
    version="0.2.3",
    reason="you may consider using *styles* instead.",
)
def paragraph(text, color=None, background_color=None, styles=None):
    """Create a styled HTML paragraphe."""
    styles = styles or {}
    if color:
        styles['color'] = color
    if background_color:
        styles['background-color'] = background_color
    html_styles = " ".join("{k}: {v};".format(k=k, v=v) for k, v in styles.items())
    html_text = xml.sax.saxutils.escape(text)
    return ('<p styles="{html_styles}">{html_text}</p>'
            .format(html_styles=html_styles, html_text=html_text))
```

In this example, we used a decorator named `@deprecated_params` to define the deprecated parameters.
Such a decorator could be implemented as follows:

```python
import functools


def deprecated_params(params, version="", reason=""):
    def decorate(func):
        @functools.wraps(func)
        def call(*args, **kwargs):
            # todo: check deprecated parameters here...
            return func(*args, **kwargs)

        return call

    return decorate
```

Advantages and disadvantages of the decorator
---------------------------------------------

Using a decorator to deprecate the use of a function parameter is interesting and offers some advantages:

- The decorator allows you to isolate the code that checks the use of this parameter from the rest of the function
  processing: Separation of Concerns design pattern.
- The decorator allows explicit documentation of the code, so the use of comments is unnecessary: “Readability matters”.
- The decorator could be used to enhance the code documentation (the docstring of the decorated function).

Disadvantages:

- The implementation of such a decorator is necessarily more complex than a specific solution.
- The resulting function (after decoration) is necessarily slower than the decorated function because of the
  introspection and parameters usage check.

Another important consideration:

- The resulting function (after decoration) must be of the same nature as the decorated function: function, method,
  static method, etc.

Generalization
--------------

In addition to the case of deprecating a parameter usage, a warning could also be issued in the following cases:

- A parameter has been renamed: the new name is more meaningful, more explicit, etc.
- One parameter is replaced by another of more general use.
- The parameter type changes, it is better to use another one.
- A parameter is considered obsolete, it will be deleted in a future version.
- A new parameter has been added, its use is strongly recommended.
- Etc.

The use cases mentioned above fall into 3 categories:

1. Deleted parameter,
2. Added parameter,
3. Modified parameter (default value, modified type or use).

The third case (modified parameter) is the most difficult to specify in general terms.

Examples of deprecations
------------------------

(This section is incomplete)

Before starting a complex implementation, it would be necessary to study how it is done, on one hand in
the [Standard Library](https://docs.python.org/3/library/index.html), and on the other hand in
the [popular libraries](https://hugovk.github.io/top-pypi-packages/) of the Open Source world such as: pip, urllib3,
boto3, six, requests, setuptools, futures, Flask, Django… At least two questions must be considered regarding
deprecation:

- How is it implemented in the source code?
- How is it documented?

Implementation Guide
--------------------

(This section is incomplete)

A parameter deprecation decorator implementation must be flexible. It is necessary to be as flexible as possible and to
use what exists for `@deprecated.basic.deprecated` and `@deprecated.sphinx.deprecated`. It is necessary to define test
cases that correspond to the most common scenarios. It is also likely that some atypical situations are clearly not
implemented. For now, we would like to continue supporting Python 2.7, as it is still used by some users.
