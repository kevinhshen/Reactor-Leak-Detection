# Optional database conversion

`file_conversion.py` converts Access `.mdb` / `.accdb` tables to CSV using an available backend: mdbtools, pyodbc with an Access driver, or access-parser.

```sh
python tools/file_conversion.py --help
```

Run this from the repository root. The help text describes the supported arguments and optional dependencies. Conversion is unnecessary when operational CSVs are already available. A converted table still needs to match the model's sensor schema and sampling interval before it can be replayed. Dose tables are not interchangeable with operational sensor data.
