# Assertions

## Always-truthy enum results

An ordinary enum member is truthy regardless of its underlying value. Assertions must check the
expected member rather than the enum result's truthiness.

```py
from enum import Enum

class Result(Enum):
    FAILURE = 0
    SUCCESS = 1

def result() -> Result:
    return Result.FAILURE

assert result()  # error: [assert-always-truthy] "Assertion is always true for type `Result`"
assert Result.FAILURE  # error: [assert-always-truthy]
assert result() is Result.SUCCESS
```

## Other always-truthy types

Functions, nonempty tuples, and final classes without truthiness methods are always truthy.

```py
from typing import final

@final
class Token: ...

def predicate() -> bool:
    return False

def check(pair: tuple[int, int], token: Token):
    assert predicate  # error: [assert-always-truthy]
    assert pair  # error: [assert-always-truthy]
    assert token  # error: [assert-always-truthy]
```

## Missing calls and awaits

An assertion of an awaitable or generator does not execute its body. Bound methods also need to be
called before their return values can be checked.

```py
async def predicate() -> bool:
    return False

class Worker:
    def ready(self) -> bool:
        return False

async def check(worker: Worker):
    assert predicate()  # error: [assert-always-truthy]
    assert await predicate()
    assert worker.ready  # error: [assert-always-truthy]
    assert worker.ready()

assert (value for value in [False])  # error: [assert-always-truthy]
```

## Conditional expression precedence

If the condition is false, a conditional expression can return a truthy value instead of performing
the intended comparison.

```py
def check(value: str):
    enabled = False
    assert value == "expected" if enabled else "fallback"  # error: [assert-always-truthy]
    assert value == ("expected" if enabled else "fallback")
```

## Unions and custom truthiness

Every alternative must be truthy. Enum mixins and custom boolean methods can allow falsy values.

```toml
[environment]
python-version = "3.11"
```

```py
from enum import Enum, IntEnum, StrEnum
from typing import Any

class First(Enum):
    VALUE = 0

class Second(Enum):
    VALUE = 0

class Integer(IntEnum):
    ZERO = 0
    ONE = 1

class String(StrEnum):
    EMPTY = ""
    TEXT = "text"

class Custom(Enum):
    VALUE = 0

    def __bool__(self) -> bool:
        return bool(self.value)

def check(
    union: First | Second,
    optional: First | None,
    integer: Integer,
    string: String,
    custom: Custom,
    dynamic: Any,
    values: list[int],
):
    assert union  # error: [assert-always-truthy]
    assert optional
    assert integer
    assert string
    assert custom
    assert dynamic
    assert values
```

## Boolean assertions

Explicit boolean checks remain useful as runtime checks, even when the type checker can establish
their result. Failing assertions are also allowed.

```py
from typing import Literal

def check(value: int, condition: bool, known: Literal[True]):
    assert condition
    assert known
    assert isinstance(value, int)
    assert value is not None
    assert True
    assert False
```

## Assertion messages

The message is still checked after reporting an always-truthy condition. The condition's type is
inferred before applying the assertion's narrowing.

```py
from enum import Enum

class Result(Enum):
    VALUE = 0

def check(value: Result | None):
    assert value
    # error: [assert-always-truthy]
    # error: [unresolved-reference]
    assert value, missing_message
```

## Condition with object that implements `__bool__` incorrectly

```py
class NotBoolable:
    __bool__: int = 3

# error: [unsupported-bool-conversion] "Boolean conversion is not supported for type `NotBoolable`"
assert NotBoolable()
```
