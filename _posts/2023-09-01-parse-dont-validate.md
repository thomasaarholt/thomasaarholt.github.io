## Parse, don't validate - In Python with Examples

**TLDR: How to use Python dataclass types to parse your data into types that give you assurance - with examples!**

There's an [excellent post](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) on using types in order to make guarantees about your data called "[Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)", from 2019. If you haven't, go read it! The code examples it uses are written in Haskell, and while that may feel strange to you, the post really is worth reading (and coming back to as you mature).

### Parse, don't validate in Python
There's another [blog post](https://stianlagstad.no/2022/05/parse-dont-validate-python-edition/) by Stian Lagstad that represents thosesame ideas in Python. In this blog post I want to extend those ideas with a fuller example to drive home how exactly this all works.

In his Python blog post, Stian shows that it might at first seem sensible to use a single class to describe *all states* that an order can be at (created, accepted, paid and shipped):

```python
@dataclass
class Order:
    id: UUID
    customer: Customer
    created_at: datetime
    accepted_at: Optional[datetime]
    paid_at: optional[datetime]
    shipped_at: Optional[datetime]
```

The issue with this is that when you write your `ship()` method/function, you need to validate that the order actually is ready for shipping: The type hinting of `paid_at: Optional[datetime]` means that we can't statically know from the type if the order has been paid yet or not - it might be `None`:

```python
def ship(self) -> Order:
    if paid_at is None: # we have to write these ifs all the time
        raise ValueError("Uh oh, order not paid yet!")
    # your shipping logic here
    self.shipped_at = datetime.now()
    return self
```
This sort of validation would be necessary in all functions that take an `Order` at the second stage onwards for a type checker to guarantee that we are shipping an order that has been paid for. This creates quite a bit of boilerplate, and it means that all `Order`s have `ship()` as a method, even if it is not yet ready for shipping!

Instead, Stian draws on the parent article and argues that it it is better to use one class for each state of the order:

```python
class ReceivedOrder:
    ...
class AcceptedOrder:
    ...
class PaidOrder:
    ...
class ShippedOrder:
    ...
```
Using these types, we can have the type checker tell us that we can't pass a `ReceivedOrder` to a `ship` function, or that the `.ship()` method doesn't exist, since it would only exist on the `PaidOrder`.

### Extending the Order example
We're going to use dataclasses for this, as they're provide a great interface for connecting data and methods in a simple wrapper. We could also use pydantic dataclasses or basemodels, which let us make further guarantees about our data, but since we're building on Stian's dataclass blog post here, we'll leave that for a future post.


Now, let's take a look at the first two classes, introduce a `Customer` class and talk about some complexity that we have to make a choice about.

```python
from __future__ import annotations # allows using not-yet-defined classes in type hints
from dataclasses import dataclass, field
from uuid import uuid4, UUID
from datetime import datetime

@dataclass
class Customer:
    "Information on the customer."
    name: str

@dataclass
class ReceivedOrder:
    customer: Customer
    id: UUID = field(default_factory=uuid4)
    created_at: datetime = field(default_factory=datetime.now)

    def accept(self) -> AcceptedOrder:
        # some acceptance logic
        return AcceptedOrder(
            id=self.id,
            customer=self.customer,
            created_at=self.created_at,
        )

@dataclass
class AcceptedOrder:
    customer: Customer
    id: UUID
    created_at: datetime
    accepted_at: datetime = field(default_factory=datetime.now)

    def paid(self) -> PaidOrder:
        # some payment logic
        ...
```

Here we create a simple class for our customer information (it only contains a name in this example). The `ReceivedOrder` contains our required fields for an order that has been received, and sets a few of them automatically by default. A UUID is an (extremely likely) unique string like `"5e976cdb-4ec9-4926-bb22-a333eb898b12"`. When setting default values like this it is important to use a "default factory", which ensures that we don't instantiate `id` or `created_at` *when we create the class*, but when we *call the class*. Otherwise *all* our instances created from the dataclass would have the same values:

```python
@dataclass
class WithoutDefaultFactory:
    timestamp: datetime = datetime.now()

print("Without default_factory")
print(WithoutDefaultFactory().timestamp) # 2023-09-05 07:04:07.790351
print(WithoutDefaultFactory().timestamp) # 2023-09-05 07:04:07.790351 (exact same!)

@dataclass
class WithDefaultFactory:
    timestamp: datetime = field(default_factory=datetime.now)

print("With default_factory")
print(WithDefaultFactory().timestamp) # 2023-09-05 07:07:10.292198
print(WithDefaultFactory().timestamp) # 2023-09-05 07:07:10.292272 (different!)
```

### An issue with inheritance and methods that no longer apply

It would have been nice to use inheritance for the AcceptedOrder:

```python
@dataclass
class AcceptedOrder(ReceivedOrder):
    accepted_at: datetime = field(default_factory=datetime.now)
    
    def paid(self) -> PaidOrder:
        ...
```

That would have left us with less repetition when defining the fields, as we would inherit the fields from `ReceivedOrder`. Unfortunately, it would also mean that a `ReceivedOrder` would be able to call `.accept()`, a call that doesn't make sense. Therefore we don't use inheritance, but write out the field names manually.

An alternative here would be to use inheritance, but write an `accept` *function* instead of a *method*. Then we nicely inherit the fields, but we lose the nice ability to autocomplete methods from our dataclasses, and could just have used [namedtuples](https://docs.python.org/3/library/collections.html#collections.namedtuple) instead. It's up to the individual on what they would prefer. This would look like the following:

```python
@dataclass
class ReceivedOrder:
    customer: Customer
    id: UUID = field(default_factory=uuid4)
    created_at: datetime = field(default_factory=datetime.now)

@dataclass
class AcceptedOrder(ReceivedOrder):
    accepted_at: datetime = field(default_factory=datetime.now)

def accept_order(order: ReceivedOrder) -> AcceptedOrder:
    "This is now a function, not a method."
    # some acceptance logic
    return AcceptedOrder(
        customer=self.customer,
        id=self.id,
        created_at=self.created_at,
    )

def paid(order: AcceptedOrder) -> "PaidOrder":
    "Also no longer a method."
    ...
```

### Following arguments that have default values with arguments without them
The eagle-eyed among you might have noticed the I switched the order of the fields `customer` and `id` from Stian's post. That is because by default, Python doesn't let you follow an argument that has a default value with an argument without one: `def foo(name: str = "Thomas", age: int): ...` is not allowed. In dataclasses, we actually have a workaround by passing `kw_only=True` to the dataclass call:

```python
@dataclass(kw_only=True)
class ReceivedOrder:
    id: UUID = field(default_factory=uuid4)
    customer: Customer
    created_at: datetime = field(default_factory=datetime.now)
```

Now all arguments have to be *named* in the `ReceivedOrder()` call, but that is fine for us. This also avoids errors when subclassing a dataclass that has a default argument, and adding a field that *doesn't* have a default value to its subclass. The `kw_only` argument was new in Python 3.10.

Nearing the end now with one final safety feature: It doesn't make sense for `AcceptedOrder` to have a default value for `created_at`. That would mean that both the received time and the accepted time would be the same (or very nearly the same). To avoid setting these values by accident we require them to be explicitly set.


Our final, fleshed out no-subclass implementation would then look like this:

### Full example

```python
from __future__ import annotations # allows using not-yet-defined classes in type hints
from dataclasses import dataclass, field
from uuid import uuid4, UUID
from datetime import datetime

@dataclass
class Customer:
    "Information on the customer."
    name: str

@dataclass(kw_only=True)
class ReceivedOrder:
    id: UUID = field(default_factory=uuid4)
    customer: Customer
    created_at: datetime = field(default_factory=datetime.now)

    def accept(self) -> AcceptedOrder:
        # some acceptance logic
        return AcceptedOrder(
            id=self.id,
            customer=self.customer,
            created_at=self.created_at,
        )

@dataclass(kw_only=True)
class AcceptedOrder:
    id: UUID
    customer: Customer
    created_at: datetime
    accepted_at: datetime = field(default_factory=datetime.now)

    def pay(self) -> PaidOrder:
        # some paying logic
        return PaidOrder(
            id=self.id,
            customer=self.customer,
            created_at=self.created_at,
            accepted_at=self.accepted_at,
        )
    
@dataclass(kw_only=True)
class PaidOrder:
    id: UUID
    customer: Customer
    created_at: datetime
    accepted_at: datetime
    paid_at: datetime = field(default_factory=datetime.now)

    def ship(self) -> ShippedOrder:
        # some shipping logic
        return ShippedOrder(
            id=self.id,
            customer=self.customer,
            created_at=self.created_at,
            accepted_at=self.accepted_at,
            paid_at=self.paid_at,
        )

@dataclass(kw_only=True)
class ShippedOrder:
    id: UUID
    customer: Customer
    created_at: datetime
    accepted_at: datetime
    paid_at: datetime
    shipped_at: datetime = field(default_factory=datetime.now)
```