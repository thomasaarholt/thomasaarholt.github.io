## Return type design pattern when returning multiples of the same type

I've been thinking about design patterns for some of my dataframe functions. I have a few functions that return a few dataframes, something like:

```python
import polars as pl
def get_data():
    df_foo = pl.DataFrame()
    df_bar = pl.DataFrame()
    return df_foo, df_bar
```
The return type of this function is `tuple[pl.DataFrame, pl.DataFrame]`. At a later date, it can be tricky to remember if you should be doing `df_foo, df_bar = get_data()` or `df_bar, df_foo = get_data()`.

I've been toying with this sort of pattern:

```python
from typing import TypeAlias
FooDF: TypeAlias = pl.DataFrame
BarDF: TypeAlias = pl.DataFrame

def get_data() -> tuple[FooDF, BarDF]:
    df_foo = pl.DataFrame()
    df_bar = pl.DataFrame()
    return df_foo, df_bar

def process_foo(df: FooDF) -> FooDF:
    return df
```

Here is what it looks like in VSCode:
![VSCode hover return type](/assets/vscode_return_type.png)

`TypeAlias` here creates a label that I can use to refer to the foo-dataframe. I sometimes have pipelines that process e.g. orders, users, routes, and so these type aliases could be repeated throughout functions that use them.

I've only just started playing with it, but it feels a bit nice, but also a bit weird.