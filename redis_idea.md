In `semantic_introspector.py`, schema introspection and pattern detection happen every time the cache is empty or expired, specifically in these sections of `sample_nodes_and_infer`:

1. **Schema Introspection:**  
   ```python
   labels_result = neo4j_conn.query("CALL db.labels()")
   labels = [row['label'] for row in labels_result]
   for label in labels:
       sample_query = f"""
       MATCH (n:`{label}`) WITH n LIMIT {self.sample_size}
       RETURN n
       """
       rows = neo4j_conn.query(sample_query)
       # ...process properties...
   ```
   This part queries Neo4j for all node labels and samples their properties—this is the slow introspection.

2. **Pattern Detection:**  
   After introspection, these methods are called:
   ```python
   self.detect_risk_patterns()
   self.detect_derived_features()
   self.detect_entity_relationships()
   ```
   These functions run additional queries and logic to find risk patterns, derived features, and entity relationships.

**Summary:**  
Whenever the cache is not available, the code runs these database queries and computations, which are slow and resource-intensive. Caching avoids repeating this work on every request.
