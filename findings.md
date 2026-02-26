# Set Exercises — Findings & Notes

Working notes on data quality issues, model decisions, and command justifications
for the Neo4j Eurovision database (Assessment 1).

---

## 1. Data Quality Issues

### 1.1 Country Name Case Mismatch

`eurovision_results.csv` uses **lowercase** country names in the `From` and `To` columns
(e.g. `albania`, `united kingdom`), whereas `Eurovision_Winners.csv` and
`eurovision_location.csv` use **proper case** (e.g. `Albania`, `United Kingdom`).

**Impact:** Country nodes created from `eurovision_results.csv` will not automatically
MERGE with Country nodes created from `Eurovision_Winners.csv`. Both sets of nodes
will exist in the database with different name values (e.g. `"Albania"` and `"albania"`
as two separate Country nodes).

**Why this is acceptable for the exercises:**
- Exercises 2 & 3 query via `[:WINNING_ENTRY]` relationships → use proper-case Country
  nodes from Winners. Unaffected.
- Exercise 3 compares `c.host_country` (proper case, from location CSV) with
  `winner.name` (proper case, from Winners). Unaffected.
- Exercise 4 queries voting results only → `r.from` and `to.name` are both lowercase,
  so comparisons are internally consistent.

**If unified Country nodes were required:** A normalisation step using `toLower()` on
all country name properties would be needed. Since Winners was imported first with
proper case, re-importing with `toLower()` applied or running a post-import normalisation
query would be necessary.

---

### 1.2 Whitespace in `Points_type` Field

The `Points_type` column in `eurovision_results.csv` contains inconsistent whitespace —
some entries have leading or trailing spaces:

| Variant                         | Count |
|---------------------------------|-------|
| `Points given`                  | 10,749 |
| `Points given by the jury`      | 2,790  |
| `Points given by televoters`    | 2,780  |
| ` Points given` (leading space) | 70     |
| `Points given ` (trailing space)| 48     |
| `Points given by the jury `     | 10     |
| `Points given by televoters `   | 10     |
| ` Points given by televoters`   | 10     |

**Resolution:** `trim(csvLine.Points_type)` is applied in the LOAD CSV command to
strip all leading and trailing whitespace before storing the value, ensuring consistent
values in the database.

---

### 1.3 Missing Years in `eurovision_results.csv`

The results file spans 68 distinct years. Two years are absent:

- **1956** — No formal points system existed in the inaugural contest.
- **2020** — The contest was cancelled due to the COVID-19 pandemic.

Both `eurovision_location.csv` and `Eurovision_Winners.csv` still contain entries for
1956 (no winner recorded; it was decided by a jury behind closed doors). 2020 is
absent from all three files since no contest took place.

**Impact:** Contest nodes for 1956 will be created from the location/winners import
but will have no `[:VOTING_RESULT]` relationships. Contest node for 2020 will not
exist at all unless explicitly created. This is expected and correct behaviour.

---

### 1.4 BOM Character in `eurovision_location.csv`

The file begins with a UTF-8 Byte Order Mark (BOM, `\uFEFF`). Neo4j's `LOAD CSV`
handles BOM characters automatically (as of Neo4j 3.x onwards), so the `Year` header
column is read correctly without manual handling.

---

## 2. Data Model Decisions

### 2.1 Contest as Central Hub Node

Contest nodes (one per year) act as the hub of the graph. The diagram provided in
the brief shows a Contest node (e.g. `1970`) at the centre, with all `Voting_Result`
relationships radiating outward to Country nodes, and one `Winning_Entry` relationship
pointing to the winning Country.

Properties stored on Contest: `year`, `host_country`, `location`.

### 2.2 `[:VOTING_RESULT]` Relationship Direction and Properties

Direction: `(Contest)-[:VOTING_RESULT]->(Country)` where Country is the **receiving**
country (the `To` field in the CSV).

The `from` field (the giving country) is stored as a property on the relationship
rather than as a separate node/relationship, consistent with the diagram showing
relationships going outward from Contest to individual Country nodes.

Properties: `from` (string), `points` (integer), `points_type` (string).

### 2.3 MERGE vs CREATE

- `MERGE` is used for Contest and Country nodes to prevent duplicate nodes if the
  import is re-run or if a Contest/Country already exists from a previous CSV load.
- `CREATE` is used for `[:VOTING_RESULT]` relationships since each voting row is
  unique; CREATE is faster than MERGE on relationships when duplicates are not a concern
  and the database is being built fresh from commands.

---

## 3. References

Neo4j, Inc. (2025) *Cypher Manual: LOAD CSV*. [Online]
Available at: https://neo4j.com/docs/cypher-manual/current/clauses/load-csv/
[Accessed: February 2026]

Neo4j, Inc. (2025) *Cypher Manual: MERGE*. [Online]
Available at: https://neo4j.com/docs/cypher-manual/current/clauses/merge/
[Accessed: February 2026]

Neo4j, Inc. (2025) *Cypher Manual: CREATE*. [Online]
Available at: https://neo4j.com/docs/cypher-manual/current/clauses/create/
[Accessed: February 2026]

Neo4j, Inc. (2025) *Cypher Manual: SET*. [Online]
Available at: https://neo4j.com/docs/cypher-manual/current/clauses/set/
[Accessed: February 2026]

Neo4j, Inc. (2025) *Cypher Manual: Scalar functions — toInteger()*. [Online]
Available at: https://neo4j.com/docs/cypher-manual/current/functions/scalar/
[Accessed: February 2026]

Neo4j, Inc. (2025) *Cypher Manual: String functions — trim()*. [Online]
Available at: https://neo4j.com/docs/cypher-manual/current/functions/string/
[Accessed: February 2026]

Neo4j, Inc. (2025) *Getting started: Import data*. [Online]
Available at: https://neo4j.com/docs/getting-started/data-import/
[Accessed: February 2026]

---

> **Note:** Verify all URLs are accessible before including in any submission.
> Documentation pages may have moved between Neo4j versions.
