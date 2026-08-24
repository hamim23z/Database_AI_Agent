# Database AI Agent

A fully local "where is this data?" tool for a Redshift warehouse with hundreds
of tables. Ask a plain-English question and it tells you which
`database.schema.table` holds that data — using semantic search that runs
entirely on your own machine.

No external AI APIs are used. The only network call in the whole workflow is
a one-time download of a small (~80MB) open-source model from Hugging Face.
After that, everything — reading your schema, computing embeddings, and
answering your questions — runs offline.

## Technology used

- **Python 3**: the script itself
- **psycopg2**: connects to Redshift and pulls schema metadata
- **Redshift system views** (`svv_all_columns`): source of table/column info
- **sentence-transformers** (`all-MiniLM-L6-v2`): local embedding model for semantic search
- **NumPy**: stores and compares vectors (cosine similarity)
- **JSON**: catalog storage format (`catalog.json`)

## How it works

1. Pull every table and column out of Redshift's own system views.
2. Write a one-line description for each table.
3. A local embedding model ([sentence-transformers](https://www.sbert.net/),
   `all-MiniLM-L6-v2`) converts each description into a vector(a list of
   numbers that captures its meaning, not just its exact words).
4. When you ask a question, it's converted to a vector the same way, and
   compared against every table's vector. The closest matches are your
   answer — even if you worded the question completely differently from the
   table's description.

## Files

| File | What it is |
|---|---|
| `redshift_schema_finder.py` | The main script |
| `catalog.json` | Generated from Redshift, then just write descriptions for each table |
| `index.npz` | Generated from `catalog.json`: the vectors used for search |

Only `redshift_schema_finder.py` is version-controlled / shared. `catalog.json`
and `index.npz` are generated locally and may contain internal schema
details, so treat them the same way you'd treat any other internal
documentation.

## Setup

### 1. Install dependencies

```bash
pip install psycopg2-binary sentence-transformers numpy
```

### 2. Set your Redshift connection details

```bash
export REDSHIFT_HOST=
export REDSHIFT_PORT=
export REDSHIFT_DB=
export REDSHIFT_USER=your_username
export REDSHIFT_PASSWORD=your_password
```

### 3. Pull the table list out of Redshift

```bash
python redshift_schema_finder.py build-catalog --database db_name
```

This queries `svv_all_columns` for every table and column in the given
database and writes `catalog.json`. Each table's `"description"` field
starts out empty.

### 4. Write a description for each table

Open `catalog.json` in any text editor and fill in `"description"` for each
table — a plain sentence describing what it holds, e.g.:

```json
{
  "database": "db_name",
  "schema": "schema_name",
  "table": "table_name",
  "description": "...",
  "columns": [ ... ]
}
```

This is the step that matters most for good search results. Use the words
people are actually likely to search with.

### 5. Build the local search index

```bash
python redshift_schema_finder.py build-index
```

The first run downloads the embedding model (~80MB, one time only). It then
embeds every table's description and columns and saves the result to
`index.npz`.

### 6. Ask questions

```bash
python redshift_schema_finder.py chat
```

```
Hamim> where is the column relating to user information?
  db_name.table.table  (match: 0.812)
  description: ...
  columns: ...
```

## Keeping it up to date

Whenever tables are added, renamed, or dropped, rerun steps 3–5:

```bash
python redshift_schema_finder.py build-catalog --database db_name
# re-add descriptions for any new tables in catalog.json
python redshift_schema_finder.py build-index
```

## Notes

- This tool only ever *locates* tables — it never runs a query against your
  data, so there's no risk of it touching production data or needing
  write/execute permissions. A read-only Redshift user is enough for
  `build-catalog`.
- Match quality depends entirely on the descriptions in `catalog.json` —
  garbage in, garbage out. Revisit descriptions for tables that aren't
  matching well.
- `redshift_schema_finder.py` uses plain password auth by default
  (`REDSHIFT_USER` / `REDSHIFT_PASSWORD`). If your environment requires
  Kerberos, swap out `get_redshift_conn()` for existing connection logic.