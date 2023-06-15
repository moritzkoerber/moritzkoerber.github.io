---
title: "Add data quality checks to your duckdb pipeline with Soda Core"
author: "Moritz Körber"
date: "15 June 2023"
categories: [Data Engineering]
tags: [Data Engineering]
---

[duckdb](https://motherduck.com/blog/six-reasons-duckdb-slaps/) is a great and fast choice for pipelines running on a single instance and is an even better match if you're pipeline oozes with SQL. To ensure that what you are doing with duckdb is actually legit, you can use the Python library [Soda Core](https://github.com/sodadata/soda-core) to run some data quality checks on your output. It's super simple to combine them. For example, you have a data pipeline that computes a data mart table/materialized view table, say in this case counted destinations. Let's simulate the underlying raw data with a pandas data frame:

```python
input_df = pandas.DataFrame(dict(destination=["New York", "New York", "Chicago"]))
```

Counting the destinations is then just a matter of

```python
duckdb.sql("SELECT destination, COUNT(destination) as cnt FROM input_df GROUP BY destination")
```

Fine, but how to check the result? Let's add some soda core checks into a `checks.yaml` – for this toy example say we check that the result features at least a single row and contains the columns "destination" and "cnt":

```yaml
checks for aggregated_destinations:
  - row_count > 0
  - schema:
      name: Confirm that required columns are present
      fail:
        when required column missing:
          - destination
          - cnt
```

When you use the in-memory database (the default), you do not need to do anything else but to create a view and run the scan using the same connection:

```python
from soda.scan import Scan

with duckdb.connect(":memory:") as con:
    con.sql(
        """\
        CREATE VIEW aggregated_destinations AS
        SELECT destination, COUNT(destination) as cnt
        FROM input_df
        GROUP BY destination
        """
    )
    scan = Scan()
    scan.add_duckdb_connection(con)
    scan.set_data_source_name("duckdb")
    scan.add_sodacl_yaml_files("soda-sql/checks.yml")
    scan.set_scan_definition_name("test_destinations")
    scan.execute()
    scan.assert_no_checks_fail()
    print(scan.get_logs_text())
```

Which prints

```
INFO   | Soda Core 3.0.38
INFO   | Scan summary:
INFO   | 2/2 checks PASSED:
INFO   |     aggregated_destinations in duckdb
INFO   |       row_count > 0 [PASSED]
INFO   |       Confirm that required columns are present [PASSED]
INFO   | All is good. No failures. No warnings. No errors.
```

Let's do a negative test and let it fail by renaming the aggregated column:

```python
with duckdb.connect(":memory:") as con:
    con.sql(
        """\
    CREATE VIEW aggregated_destinations AS
    SELECT destination, COUNT(destination) as counted_destinations
    FROM input_df
    GROUP BY destination
    """
    )
    scan = Scan()
    scan.add_duckdb_connection(con)
    scan.set_data_source_name("duckdb")
    scan.add_sodacl_yaml_files("soda-sql/checks.yml")
    scan.set_scan_definition_name("test_destinations")
    scan.execute()
    scan.assert_no_checks_fail()
    print(scan.get_logs_text())
```

This throws `AssertionError: Check results failed: [schema] FAIL (fail_missing_column_names = [cnt], schema_measured = [destination varchar, counted_destinations bigint])` – as expected.

When you have a database file though, you need to use the datasource config in the form of a `configuration.yaml`:

```yaml
data_source my_data:
  type: duckdb
  path: "database.db"
```

And then the same show as before:

```python
with duckdb.connect("database.db") as con:
    con.sql(
        """\
        CREATE VIEW aggregated_destinations AS
        SELECT destination, COUNT(destination) as cnt
        FROM input_df
        GROUP BY destination
        """
    )
    scan = Scan()
    scan.add_duckdb_connection(con)
    scan.set_data_source_name("my_data")
    scan.add_configuration_yaml_file("soda-sql/configuration.yml")
    scan.add_sodacl_yaml_files("soda-sql/checks.yml")
    scan.set_scan_definition_name("test_destinations")
    scan.execute()
    scan.assert_no_checks_fail()
    print(scan.get_logs_text())
```

Don't let data quality issues drive your quackers! 🦆
