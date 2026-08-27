## What it does

Checks for assertions of non-boolean values whose inferred types are always truthy.

## Why is this bad?

An assertion of an always-truthy value cannot detect an incorrect result. For example,
ordinary enum members are truthy even when their underlying value is zero. Asserting an
enum result therefore does not check which member was returned.

This can also indicate a missing function call or an assertion of a nonempty tuple instead
of its contents. Compare against the expected value or check the intended property explicitly.

Values of type `bool` are exempt, including `Literal[True]`, because
they can be intentional runtime checks of properties that are also expressed in type annotations.
Types with unknown truthiness, including `Any`, are not reported.

## Example

```python
from enum import Enum


class Result(Enum):
    FAILURE = 0
    SUCCESS = 1


def run() -> Result:
    return Result.FAILURE


assert run()  # error
```

Use instead:

```python
assert run() is Result.SUCCESS
```
