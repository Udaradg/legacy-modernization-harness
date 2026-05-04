# Neo4j import instructions — Legacy Modernization Harness (COBOL)

## Option A — Cypher script (recommended, easiest)

1. Open Neo4j Desktop and start your database
2. Click "Open Neo4j Browser"
3. Create a new database named `cobol` (or use the default)
4. In the Browser, click the folder icon to open a file
5. Select `output/final_report/graph/neo4j/import.cypher`
6. Click "Run" — the script loads all nodes and relationships
7. When complete, run this to verify:
   ```cypher
   MATCH (n) RETURN labels(n)[0] AS type, count(n) AS count
   ORDER BY count DESC
   ```

## Option B — CSV import (for large codebases >50,000 nodes)

1. Stop your Neo4j database
2. Copy all files from `output/final_report/graph/neo4j/nodes/` and
   `output/final_report/graph/neo4j/rels/` into your Neo4j import directory:
   - macOS/Linux: `~/Library/Application Support/Neo4j Desktop/.../import/`
   - Windows: `%APPDATA%\Neo4j Desktop\...\import\`
3. Open a terminal and run:
   ```
   neo4j-admin database import full \
     --nodes=Program=import/programs.csv \
     --nodes=Copybook=import/copybooks.csv \
     --nodes=JclJob=import/jcl_jobs.csv \
     --nodes=File=import/files.csv \
     --nodes=Record=import/records.csv \
     --nodes=Field=import/fields.csv \
     --nodes=Paragraph=import/paragraphs.csv \
     --nodes=BusinessRule=import/business_rules.csv \
     --nodes=RuleSet=import/rule_sets.csv \
     --nodes=Condition=import/conditions.csv \
     --relationships=CALLS=import/calls.csv \
     --relationships=COPIES=import/copies.csv \
     --relationships=EXECUTES=import/executes.csv \
     --relationships=DEFINES=import/defines.csv \
     --relationships=CONTAINS=import/contains_prog_para.csv \
     --relationships=CONTAINS=import/contains_rec_field.csv \
     --relationships=ENFORCED_IN=import/enforced_in.csv \
     --relationships=TESTS=import/tests.csv \
     --relationships=HAS_CONDITION=import/has_condition.csv \
     --relationships=RELATED_TO=import/related_to.csv \
     --database=cobol --overwrite-destination
   ```
4. Start the database and open Neo4j Browser

## What was imported

| Node type     | Count |
|---------------|-------|
| Program       | 38    |
| Copybook      | 20    |
| JclJob        | 15    |
| Record        | 706   |
| Field         | 430   |
| Paragraph     | 379   |
| BusinessRule  | 90    |
| Condition     | 114   |

| Relationship        | Count |
|---------------------|-------|
| CALLS               | 19    |
| COPIES              | 30    |
| EXECUTES            | 14    |
| DEFINES             | 706   |
| CONTAINS (prog→para)| 379   |
| CONTAINS (rec→field)| 430   |
| ENFORCED_IN         | 184   |
| TESTS               | 31    |
| HAS_CONDITION       | 3     |
| RELATED_TO          | 38    |
| **Total**           | **1,834** |

## First queries to run after import

See `cypher_library.md` for the full analytical query catalogue (25 queries).

Quick verification:
```cypher
-- Count all nodes by type
MATCH (n) RETURN labels(n)[0] AS type, count(n) AS count ORDER BY count DESC;

-- Top most-called programs
MATCH (a:Program)-[:CALLS]->(b:Program)
WITH b, count(a) AS callers
RETURN b.program_id, callers ORDER BY callers DESC LIMIT 10;

-- All business rules with high confidence
MATCH (r:BusinessRule {confidence:'high'})
RETURN r.rule_id, r.name, r.category ORDER BY r.category;
```
