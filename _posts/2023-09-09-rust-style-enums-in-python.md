# Rust style enums in Python

## TL;DR

Implementation of rust style enums in Python: [rust_enum](https://github.com/girvel/rust_enum)

## Encountering the issue

Today I was working on an AI module in my pet gamedev project and encountered a problem with magic constants. Look:

```py
def try_producing_target(self, subject, perception):
    if self.subject is None: return False  # what does False mean?
    if abs2(sub2(subject.p, self.subject.p)) <= self.d: return None  # what is the difference with None?

    if self.period.step():
        return self.subject.p
    return False  # False again, wtf?
```

This method is the implementation of the AI component, that allows an entity to switch path and follow some other entity. If there is no subject to be followed, no change in path target is needed; if the target is close, we need to stop any movement; if everything is okay, we need to switch the target of the path to followed entity's position each N ticks.

The only problem with this code is the return values. What does False mean? What does None mean? These are just some magic values that have no meaning for the reader. And on the receiving end, it does not look any better:

```py
def make_decision(self, subject, perception):
    # noinspection PySimplifyBooleanCheck
    if (target := self.follower.try_producing_target(subject, perception)) != False: self.pather.going_to = target
    if (action := self.pather.try_going(subject, perception)) is not None: return action
```

This is a fairly simple method in a small module, so it is nothing to worry about yet, but I am planning to grow it, and it is reasonable to assume that I will encounter the same problem again. To solve it, ideally I need to define the set of possible return values with self-explanatory names. And the perfect solution for that is rust-style enums:

```rust
enum TargetChange {
  Nothing,
  To(Option<int2>),
}
```

When we set the return type of the `try_producing_target` to such an enum, it tells the programmer and the compiler that it returns either that there is no target change needed or that we need to change the target to some position. No magic values, only meaningful constants grouped in one enum. Also, it can not return any other kind of value and the returned result can easily be `match`ed. This is very readable, and it is the best solution for such a class of problems, so there should be a library that does the same in Python, yes?

## Building the solution

A quick google search reveals that no, there are none, so I took matters into my own hands. Rust-style enum should probably use class syntax and be marked with a decorator telling that it is an enum, so something like this:

```py
@enum
class TargetChange:
    Nothing = {}
    To = {"target": Optional[int2]}
```

The easiest way to enable correct pattern matching is to dynamically create dataclasses from every non-dunder member of the class:

```py
from dataclasses import make_dataclass


def enum(cls):
    for field_name in dir(cls):
        if field_name.startswith('__') and field_name.endswith('__'): continue
        setattr(cls, field_name, make_dataclass(field_name, list(getattr(cls, field_name).items()), bases=(cls, )))
    return cls
```

It works, but sadly the `TargetChange.To(...)` raises warnings as the linter thinks that we are trying to call the dictionary; also, creating dataclasses from every non-dunder member makes adding additional attributes and methods to the enum impossible. So, new syntax:

```py
@enum
class TargetChange:
    Nothing = Case()
    To = Case(target=Optional[int2])
```

And better implementation:

```py
from dataclasses import make_dataclass


def enum(cls):
    for field_name in dir(cls):
        if not isinstance((value := getattr(cls, field_name)), Case): continue
        setattr(cls, field_name, make_dataclass(field_name, list(value.dict.items()), bases=(cls, )))
    return cls


class Case:
    def __init__(self, **attributes):
        self.dict = attributes

    # to disable warnings
    def __call__(self, *args, **kwargs):
        pass

```

And 17 lines of code is basically all that is needed to port Rust enums into Python.

## The final result

Module for following:

```py
@enum
class TargetChange:
    Nothing = Case()
    To = Case(target=Optional[int2])
```

```py
def try_producing_target(self, subject, perception) -> TargetChange:
    if self.subject is None or self.subject.p not in perception.vision.physical: return TargetChange.Nothing()
    if abs2(sub2(subject.p, self.subject.p)) <= self.d: return TargetChange.To(None)

    if self.period.step():
        return TargetChange.To(self.subject.p)
    return TargetChange.Nothing()
```

And its usage:

```py
match self.follower.try_producing_target(subject, perception):
    case TargetChange.To(p): self.pather.going_to = p
```

Problem solved.

## P.S.

I pushed the library to [PyPI](https://pypi.org/project/rust-enum/) and [GitHub](https://github.com/girvel/rust_enum). You can use it if you want to. Also, kind of thinking about implementing Rust-like Result and Option, they have some useful methods.
