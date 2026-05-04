# Cypher query library — Legacy Modernization Harness (COBOL)

Paste any query into Neo4j Browser and press Ctrl+Enter to run.

---

## Program topology

### Q01 — Most connected programs (hubs)
```cypher
MATCH (caller:Program)-[:CALLS]->(hub:Program)
WITH hub, count(DISTINCT caller) AS caller_count
ORDER BY caller_count DESC LIMIT 10
RETURN hub.program_id, hub.subtype, caller_count, hub.rule_count;
```

### Q02 — Full call chain from a JCL job
```cypher
MATCH path = (j:JclJob {job_name: 'RPTAUD00'})
             -[:EXECUTES]->(:Program)-[:CALLS*0..10]->(:Program)
RETURN [n IN nodes(path) | coalesce(n.job_name, n.program_id)] AS chain,
       length(path) AS depth ORDER BY depth;
```

### Q03 — Programs with no callers (entry points)
```cypher
MATCH (p:Program)
WHERE NOT (:Program)-[:CALLS]->(p)
  AND NOT (:JclJob)-[:EXECUTES]->(p)
RETURN p.program_id, p.subtype, p.rule_count;
```

### Q04 — Programs with unstructured flow
```cypher
MATCH (p:Program)
WHERE p.has_unstructured_flow = true OR p.has_alter = true
RETURN p.program_id, p.has_alter, p.has_unstructured_flow, p.rule_count
ORDER BY p.has_alter DESC;
```

### Q05 — Circular call chains
```cypher
MATCH path = (a:Program)-[:CALLS*2..]->(a)
RETURN [n IN nodes(path) | n.program_id] AS cycle, length(path) AS depth;
```

---

## Copybook and data impact

### Q06 — Most widely shared copybooks
```cypher
MATCH (c:Copybook)<-[:COPIES]-(p:Program)
WITH c, count(p) AS usage, collect(p.program_id) AS programs
ORDER BY usage DESC LIMIT 15
RETURN c.copybook_id, c.total_fields, usage, programs;
```

### Q07 — Impact if copybook changes (replace ERRHAND)
```cypher
MATCH (c:Copybook {copybook_id: 'ERRHAND'})<-[:COPIES]-(p:Program)
OPTIONAL MATCH (p)-[:CONTAINS]->(para:Paragraph)<-[:ENFORCED_IN]-(r:BusinessRule)
RETURN p.program_id,
       count(DISTINCT para) AS paragraphs_affected,
       count(DISTINCT r) AS rules_affected
ORDER BY rules_affected DESC;
```

### Q08 — High-shared fields (used by 5+ programs)
```cypher
MATCH (f:Field {shared_level: 'high_shared'})
MATCH (r:Record)-[:CONTAINS]->(f)
MATCH (p:Program)-[:DEFINES]->(r)
WITH f, count(DISTINCT p) AS program_count WHERE program_count >= 5
RETURN f.field_name, f.normalized_type, f.length, program_count
ORDER BY program_count DESC;
```

### Q09 — Where is a field defined? (replace WS-FILE-STATUS)
```cypher
MATCH (f:Field {field_name: 'WS-FILE-STATUS'})
MATCH (r:Record)-[:CONTAINS]->(f)
RETURN f.field_id, f.normalized_type, f.length, r.record_id;
```

### Q10 — REDEFINES groups (alternative layouts)
```cypher
MATCH (alias:Field)-[rel:REDEFINES]->(anchor:Field)
MATCH (r:Record)-[:CONTAINS]->(anchor)
RETURN r.record_id,
       anchor.field_name AS storage_anchor,
       collect(alias.field_name) AS aliases,
       anchor.length AS bytes;
```

---

## Business rules

### Q11 — All rules for a program (replace PORTMSTR)
```cypher
MATCH (p:Program {program_id: 'PORTMSTR'})
      -[:CONTAINS]->(para:Paragraph)<-[:ENFORCED_IN]-(r:BusinessRule)
RETURN r.rule_id, r.name, r.category, r.confidence,
       para.name AS paragraph, r.requires_sme_review
ORDER BY r.confidence DESC;
```

### Q12 — Rules that test a specific field (replace WS-FILE-STATUS)
```cypher
MATCH (r:BusinessRule)-[rel:TESTS]->(f:Field {field_name: 'WS-FILE-STATUS'})
RETURN r.rule_id, r.name, r.category, r.confidence, rel.role;
```

### Q13 — Rules needing SME review by program
```cypher
MATCH (r:BusinessRule {requires_sme_review: true})
      -[:ENFORCED_IN]->(para:Paragraph)<-[:CONTAINS]-(p:Program)
WITH p, collect({rule: r.rule_id, name: r.name}) AS rules
RETURN p.program_id, size(rules) AS review_count, rules
ORDER BY review_count DESC;
```

### Q14 — Duplicated rules across programs
```cypher
MATCH (r:BusinessRule {is_duplicated: true})
      -[:ENFORCED_IN]->(para:Paragraph)<-[:CONTAINS]-(p:Program)
WITH r, collect(DISTINCT p.program_id) AS programs
RETURN r.rule_id, r.name, r.category, size(programs) AS count, programs
ORDER BY count DESC;
```

### Q15 — Rules by category and confidence
```cypher
MATCH (r:BusinessRule)
RETURN r.category, r.confidence, count(r) AS rule_count
ORDER BY r.category, r.confidence;
```

---

## File and I/O analysis

### Q16 — File access map
```cypher
MATCH (p:Program)-[rel:READS|WRITES]->(f:File)
RETURN f.file_name, f.file_type,
       collect({program: p.program_id, access: type(rel)}) AS accessors;
```

### Q17 — Files written by only one program (safe to refactor)
```cypher
MATCH (p:Program)-[:WRITES]->(f:File)
WITH f, count(p) AS writer_count WHERE writer_count = 1
MATCH (w:Program)-[:WRITES]->(f)
RETURN f.file_name, w.program_id AS sole_writer;
```

### Q18 — Producer-consumer file pairs
```cypher
MATCH (producer:Program)-[:WRITES]->(f:File)<-[:READS]-(consumer:Program)
WHERE producer <> consumer
RETURN f.file_name, producer.program_id AS producer, consumer.program_id AS consumer;
```

---

## Complexity and risk

### Q19 — Highest complexity paragraphs
```cypher
MATCH (p:Program)-[:CONTAINS]->(para:Paragraph)
WHERE para.complexity_score IS NOT NULL
RETURN p.program_id, para.name, para.complexity_score,
       para.has_goto, para.has_call
ORDER BY para.complexity_score DESC LIMIT 20;
```

### Q20 — Dead code candidates
```cypher
MATCH (para:Paragraph {is_dead_code: true})<-[:CONTAINS]-(p:Program)
OPTIONAL MATCH (para)<-[:ENFORCED_IN]-(r:BusinessRule)
RETURN p.program_id, para.name, para.start_line,
       count(r) AS rules_in_dead_code;
```

### Q21 — Programs at highest modernisation risk
```cypher
MATCH (p:Program)
OPTIONAL MATCH (:Program)-[:CALLS]->(p)
WITH p, count(*) AS caller_count
RETURN p.program_id,
       p.has_unstructured_flow AS goto_risk,
       p.has_alter AS alter_risk,
       p.rule_count AS business_rules,
       caller_count AS called_by,
       (CASE WHEN p.has_alter THEN 3 ELSE 0 END +
        CASE WHEN p.has_unstructured_flow THEN 2 ELSE 0 END +
        CASE WHEN caller_count > 5 THEN 2 ELSE 0 END +
        CASE WHEN p.rule_count > 10 THEN 1 ELSE 0 END) AS risk_score
ORDER BY risk_score DESC LIMIT 15;
```

### Q22 — 88-level coded values for a field (replace PREREQS-SATISFIED)
```cypher
MATCH (f:Field {field_name: 'PREREQS-SATISFIED'})-[:HAS_CONDITION]->(c:Condition)
RETURN f.field_name, c.condition_name, c.values_raw;
```

---

## Cross-cutting

### Q23 — End-to-end: job → rules enforced
```cypher
MATCH (j:JclJob {job_name: 'PORTTEST'})
      -[:EXECUTES]->(:Program)-[:CALLS*0..5]->(p:Program)
      -[:CONTAINS]->(para:Paragraph)<-[:ENFORCED_IN]-(r:BusinessRule)
RETURN DISTINCT j.job_name, p.program_id, para.name AS paragraph,
       r.rule_id, r.name, r.category ORDER BY p.program_id;
```

### Q24 — Impact of changing a field type (replace WS-FILE-STATUS)
```cypher
MATCH (f:Field {field_name: 'WS-FILE-STATUS'})
MATCH (r:Record)-[:CONTAINS]->(f)
MATCH (p:Program)-[:DEFINES]->(r)
OPTIONAL MATCH (rule:BusinessRule)-[:TESTS]->(f)
RETURN f.field_name, f.normalized_type,
       collect(DISTINCT p.program_id) AS programs_to_change,
       collect(DISTINCT rule.rule_id) AS rules_to_review;
```

### Q25 — Rules sharing the same field (related rules cluster)
```cypher
MATCH (a:BusinessRule)-[rel:RELATED_TO]->(b:BusinessRule)
WHERE rel.relationship_type = 'SAME_FIELD'
RETURN a.rule_id, a.category, b.rule_id, b.category, rel.relationship_type
ORDER BY a.rule_id LIMIT 25;
```
