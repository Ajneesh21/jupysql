---
jupytext:
  notebook_metadata_filter: myst
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.14.5
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
myst:
  html_meta:
    description lang=en: Recipes for JupySQL
    keywords: jupyter, sql, jupysql
    property=og:locale: en_US
---

```{code-cell} ipython3
:tags: [remove-cell]

# clean up all .db files (this cell will not be displayed in the docs)
from pathlib import Path
from glob import glob

for file in (Path(f) for f in glob("*.db")):
    if file.exists():
        print(f"Deleting: {file}")
        file.unlink()
```

# How-To

## Query CSV files with SQL

You can use `JupySQL` and `DuckDB` to query CSV files with SQL in a Jupyter notebook.

+++

### Installation

```{code-cell} ipython3
%pip install jupysql duckdb duckdb-engine --quiet
```

### Setup

Load JupySQL:

```{code-cell} ipython3
%load_ext sql
```

Create an in-memory DuckDB database:

```{code-cell} ipython3
%sql duckdb://
```

Download some sample data:

```{code-cell} ipython3
from urllib.request import urlretrieve

_ = urlretrieve(
    "https://raw.githubusercontent.com/mwaskom/seaborn-data/master/penguins.csv",
    "penguins.csv",
)
```

### Query

```{code-cell} ipython3
%%sql
SELECT *
FROM penguins.csv
LIMIT 3
```

```{code-cell} ipython3
%%sql
SELECT species, COUNT(*) AS count
FROM penguins.csv
GROUP BY species
ORDER BY count DESC
```

## Convert to `polars.DataFrame`

```{code-cell} ipython3
%%sql results <<
SELECT species, COUNT(*) AS count
FROM penguins.csv
GROUP BY species
ORDER BY count DESC
```

```{code-cell} ipython3
import polars as pl
```

```{code-cell} ipython3
pl.DataFrame((tuple(row) for row in results), schema=results.keys)
```

## Register SQLite UDF

To register a user-defined function (UDF) when using SQLite, you can use [SQLAlchemy's `@event.listens_for`](https://docs.sqlalchemy.org/en/14/dialects/sqlite.html#user-defined-functions) and SQLite's [`create_function`](https://docs.python.org/3/library/sqlite3.html#sqlite3.Connection.create_function):

### Install JupySQL

```{code-cell} ipython3
%pip install jupysql --quiet
```

### Create engine and register function

```{code-cell} ipython3
from sqlalchemy import create_engine
from sqlalchemy import event


def mysum(x, y):
    return x + y


engine = create_engine("sqlite://")


@event.listens_for(engine, "connect")
def connect(conn, rec):
    conn.create_function(name="MYSUM", narg=2, func=mysum)
```

### Create connection with existing engine

```{versionadded} 0.5.1
Pass existing engines to `%sql`
```

```{code-cell} ipython3
%load_ext sql
```

```{code-cell} ipython3
%sql engine
```

## Query

```{code-cell} ipython3
%%sql
SELECT MYSUM(1, 2)
```

## Connect to a SQLite database with spaces

Currently, due to a limitation in the argument parser, it's not possible to directly connect to SQLite databases whose path contains spaces; however, you can do it by creating the engine first.

### Setup

```{code-cell} ipython3
%pip install jupysql --quiet
```

```{code-cell} ipython3
%load_ext sql
```

## Connect to db

```{code-cell} ipython3
from sqlalchemy import create_engine

engine = create_engine("sqlite:///my database.db")
```

Add some sample data:

```{code-cell} ipython3
import pandas as pd

_ = pd.DataFrame({"x": range(5)}).to_sql("numbers", engine)
```

```{code-cell} ipython3
%sql engine
```

```{code-cell} ipython3
%%sql
SELECT * FROM numbers
```

## Switch connections

```{versionadded} 0.5.2
`-A/--alias`
```

```{code-cell} ipython3
from sqlalchemy import create_engine
import pandas as pd

engine_one = create_engine("sqlite:///one.db")
pd.DataFrame({"x": range(5)}).to_sql("one", engine_one)

engine_two = create_engine("sqlite:///two.db")
_ = pd.DataFrame({"x": range(5)}).to_sql("two", engine_two)
```

```{code-cell} ipython3
%load_ext sql
```

Assign alias to both connections so we can switch them by it:

```{code-cell} ipython3
%sql sqlite:///one.db --alias one
%sql sqlite:///two.db --alias two
```

```{code-cell} ipython3
%sql
```

Pass the alias to use such connection:

```{code-cell} ipython3
%%sql one
SELECT * FROM one
```

It becomes the current active connection:

```{code-cell} ipython3
%sql
```

Hence, we can skip it in upcoming queries:

```{code-cell} ipython3
%%sql
SELECT * FROM one
```

Switch connection:

```{code-cell} ipython3
%%sql two
SELECT * FROM two
```

```{code-cell} ipython3
%sql
```

Close by passing the alias:

```{code-cell} ipython3
%sql --close one
```

```{code-cell} ipython3
%sql
```

```{code-cell} ipython3
%sql --close two
```

```{code-cell} ipython3
%sql -l
```

## Connect to existing `engine`

Pass the name of the engine:

```{code-cell} ipython3
some_engine = create_engine("sqlite:///some.db")
```

```{code-cell} ipython3
%sql some_engine
```

## Use `%sql`/`%%sql` in Databricks

Databricks uses the same name (`%sql`/`%%sql`) for its SQL magics; however, JupySQL exposes a `%jupysql`/`%%jupysql` alias so you can use both:

```{code-cell} ipython3
%jupysql duckdb://
```

```{code-cell} ipython3
%jupysql SELECT * FROM "penguins.csv" LIMIT 3
```

```{code-cell} ipython3
%%jupysql
SELECT *
FROM "penguins.csv"
LIMIT 3
```

## Ignore deprecation warnings

We display warnings to let you know when the API will change so you have enough time to update your code, if you want to supress this warnings, add this at the top of your notebook:

```{code-cell} ipython3
import warnings

warnings.filterwarnings("ignore", category=FutureWarning)
```
