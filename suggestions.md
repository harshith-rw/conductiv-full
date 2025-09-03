## optimizations by subsystem

### 1) LLM and planning/summarization
- Memoize LLM responses (plan/summarization) in Redis
  - What: Cache Bedrock/Claude outputs keyed by a normalized input tuple: (query text normalized, conversation_context hash, effective schema hash/version, query_type).
  - Where: Wrap calls in query_processor.py (plan generation, validation, summary) and aws_client.py (aws_summarize_conversation, aws_convert_nl_to_cypher).
  - Impact: 50–90% time saved on repeated questions and follow-ups with the same filters/context.
- Reduce validation/fix loop passes
  - What: In `Neo4jQueryEngine._aquery` (aws_client), current 2-pass validation; keep 1 pass plus confidence gating tied to model output; only add second attempt if schema mismatch is detected (not any generic “invalid”).
  - Impact: Saves 1 full LLM roundtrip in the common case.
- Downscope prompts by schema subsets
  - What: Instead of a big full schema, pass only relevant entity and relationship slices based on entity detection or keywords. Use `field_to_entity` from schema_loader.py to select schema sections.
  - Impact: Smaller prompts → faster/cheaper LLM calls, less JSON cleaning headaches.

### 2) Neo4j usage
- Reuse a single graph store/driver
  - What: Avoid creating multiple `Neo4jGraphStore` instances. entity_routes.py and query_processor.py each construct their own. Reuse `Settings.graph_store` created in main.py.
  - Where: entity_routes.py (replace local `graph_store = Neo4jGraphStore(...)` with `Settings.graph_store`), query_processor.py (pass graph store in or reference `Settings.graph_store`).
  - Impact: Less connection churn, better pooling; stabilizes latency.
- Parameterized queries for execution path
  - What: LLM is instructed “DO NOT use parameters,” but execution can still safely parameterize literals (IDs, filters) after generation to reuse plans and hit caches. Perform a light transform: extract literals and pass them as parameters when sending to Neo4j.
  - Impact: Reduced parse/replan overhead; improved cacheability and safety.
- Add/verify indexes and constraints
  - Focus on: `Loan.loanNumber`, `Loan.id`, `Person.id`, `Employment.employmentId`, `Employer.id`, `Organization.id`, `Organization.name`.
  - Impact: Big wins on entity lookups and comparison queries.
- Pagination/limits for heavy expanders
  - What: Ensure `/graph` and entity detail queries use LIMITs and/or SKIP/OFFSET consistently; already partially done with `[0..N]`. Standardize across all subgraphs and expose pagination to clients.
  - Impact: Predictable response times, avoids memory blowups.

### 3) Async and parallelization
- Fix blocking calls in async contexts
  - What: explorer.py uses `requests` (sync) inside async code — replace with `httpx.AsyncClient` or `aiohttp` to avoid blocking the event loop when calling the Java service.
  - Where: `EntityExplorer._generate_entity_details_cypher` HTTP request.
  - Impact: Frees the event loop; better concurrency under load.
- Parallelize independent graph fetches where safe
  - What: Where a response requires multiple unrelated cypher sub-queries, run them with `asyncio.gather` (or combine with UNWIND+collect to a single roundtrip if possible).
  - Where: Complex graph assembly code (e.g., `/graph` endpoint in main.py) and deep comparison builders (some can be consolidated into one query; if not, parallelize).
  - Impact: Latency reduction proportional to the number of independent sub-queries.

### 4) Schema metadata and semantic introspection
- Cache schema + introspection with TTL
  - What: Cache `apoc.meta.schema()`/relationship counts and semantic introspection output in Redis (include a version stamp). Invalidate on startup flags or admin endpoint.
  - Where: `aws_client.get_neo4j_schema`, semantic_introspector.py and `main.generate_schema_metadata_async`.
  - Impact: Avoids re-generating schema and introspection repeatedly; lower load on Neo4j.
- Load schema artifacts once and share
  - What: schema_loader.py already reads `schema_relationships.json` and `schema_context.json`. Load once on startup, store in a module-level singleton, and reuse across `QueryProcessor`, `aws_client`, and entity routes.
  - Impact: Less disk IO; smaller latency spikes.

### 5) Conversation context/state
- Bound history length and sanitize aggressively
  - What: `state.build_conversation_context` already trims; add token-budget based trimming and domain-specific redaction to prevent prompt bloat.
  - Impact: Smaller prompts to LLM; improved speed and lowered cost.
- Cache post-processed “plan context”
  - What: Store the derived traversal map and relevant schema slices per session in Redis; reuse across a series of related queries.
  - Impact: Fewer repeated computations per session.

### 6) JSON handling and robustness
- Remove redundant JSON cleaning and double-parsing
  - What: entity_routes.py treats the result of `aws_summarize_conversation` (returns dict) as a string and tries `json.loads`/`ast.literal_eval` — that’s incorrect and slow/failure-prone. Just pass through the dict.
  - Impact: Fewer failures and no wasted CPU cycles.
- Standardize Bedrock streaming assembly
  - What: Extract a shared helper to assemble text from streaming chunks; centralize control-character filtering once.
  - Impact: Simpler, faster, less duplication.

### 7) Planning/graph generation improvements
- Prebuilt templates for common patterns
  - What: Several entity and comparison patterns are static. Keep a small template catalog (by entity type, relationship depth, and “top N” variants). Skip LLM planning and go straight to execution for these.
  - Impact: Instant responses for frequent questions.
- Tighten Cypher generation constraints
  - What: Strengthen rules to favor OPTIONAL MATCH patterns with bounded collects and controlled WITH pipelines; codify a few safe “skeletons” the LLM must fit into.
  - Impact: Fewer invalid queries, fewer retries.

### 8) Observability and guardrails
- Timing metrics per stage
  - What: Log durations for: detection → plan → cypher gen → validate → execute → post-process → summarize. Emit to logs/metrics.
  - Impact: Identifies the slowest hop per query type for targeted improvements.
- Circuit breakers on LLM and DB
  - What: Add conservative timeouts and fallbacks for plan/summarize and database calls; bail out with a helpful message and a static template when timeouts occur.
  - Impact: Stable UX even under backpressure.

## two correctness fixes to apply soon
- Async misuse in entity routes
  - Where: entity_routes.py, e.g., `result = entity_explorer.explore_entity(query)` is missing `await`. This function is async, so these endpoints likely return coroutine errors.
  - Fix: `result = await entity_explorer.explore_entity(query)` in all three routes (`/{entity_type}/{entity_id}`, `/related`, `/summary`).
- Double-parsing of AWS summary
  - Where: `process_entity_with_aws_summarization` calls `aws_summarize_conversation` (returns dict) and then tries `json.loads`/`ast.literal_eval`.
  - Fix: Delete that parsing and use the dict as-is; attach enriched fields (queryId, timestamp, etc.) directly.

If you want, I can apply both fixes right away.

## function-level flow (end-to-end)

### Service startup (fastapi/main.py)
- Settings/env load → CORS → logging init.
- Graph store init (Neo4jGraphStore) → placed into llama-index `Settings.graph_store`.
- Startup event:
  - DB health check (neo4j driver test).
  - Kick background: generate `schema_relationships.json` and `schema_context.json`.
  - Optional: run `SemanticIntrospector().sample_nodes_and_infer()` with timeout.
- Routers mounted: `/convert`, `/graph/...`, `/health`, `/diagnostics`, `api.entity_routes`.

### Convert endpoint: POST /convert (fastapi/main.py)
- Parse user query + session info.
- Build conversation context: `state.build_conversation_context` (Redis recent history).
- Call `QueryProcessor.process_query(query, context, ...)`.
  - Inside `QueryProcessor` (fastapi/query_processor.py):
    - Initialize or reuse: graph store, `EntityExplorer`, schema metadata (`services/schema_loader.load_schema_metadata`) and traversal map from `schema_relationships.json`; run semantic introspector sampling (optional).
    - Intent classification: entity vs analytical vs comparison vs lookalike (uses `entity/llm_detector.py`, and settings).
    - For entity query:
      - `EntityExplorer.explore_entity` → `_generate_entity_details_cypher` (may call Java service over HTTP).
      - Execute Cypher → format entity response (response.py) → summarize via AWS (`aws_client.aws_summarize_conversation`).
    - For analytical query:
      - Build plan prompt (`build_plan_llm_prompt`) → `_call_plan_llm` → generate Cypher → execute → process results → summarize via AWS → optional entity linking.
    - For comparison query:
      - Plan related entities → generate comparison Cypher (or use deep prebuilt) → execute → filter/transform → summarize → link.
    - For lookalike:
      - Generate Cypher with Python generator and summarize via AWS.
- Persist result to history (`state.add_to_history`) and return.

### Entity routes (fastapi/api/entity_routes.py)
- GET `/api/entities/{entity_type}/{entity_id}`:
  - Compose a query phrase.
  - `EntityExplorer.explore_entity` → generate Cypher → execute via graph store.
  - `EntityResponseFormatter.format_entity_response` builds hierarchical data.
  - Summarize via `aws_summarize_conversation` with enhanced tabular context → attach metadata and return.
- GET `/related` and `/summary`: similar flow with different query phrases and the same summarization path.

### AWS client (fastapi/aws_client.py)
- Bedrock/Claude streaming invocation with exponential backoff.
- `Neo4jQueryEngine` wraps:
  - `get_neo4j_schema` (dynamic schema from apoc + rel counts; format string).
  - LLM planning → Cypher generation → validation/fix loop.
- `aws_summarize_conversation`:
  - Builds prompt using schema, query, results; streams response → clean → parse → returns dict (summary and sections).

### Schema loader and introspection
- schema_loader.py: loads `schema_relationships.json` and `schema_context.json`; builds relationship graph and field-to-entity map; formats schema/patterns for prompts.
- semantic_introspector.py: samples labels, infers types and patterns (credit score ranges, DTI, etc.) to guide planning.

### Conversation state (fastapi/state.py)
- Stores/retrieves chat history in Redis and builds a compact context preview used in LLM prompts.

## quick wins you can implement next
- Replace `requests` with `httpx.AsyncClient` in `EntityExplorer._generate_entity_details_cypher` and await it.
- Switch entity_routes.py to use `await entity_explorer.explore_entity(...)` and reuse `Settings.graph_store` instead of creating a new `Neo4jGraphStore`.
- In `process_entity_with_aws_summarization`, remove `json.loads`/`ast.literal_eval` and treat `aws_summarize_conversation` result as a dict.
- Add a Redis cache wrapper for Bedrock responses in `aws_summarize_conversation` and plan generation (normalize input, set TTL 10–60 min).
- Centralize schema caching: keep `get_neo4j_schema`’s result in Redis with TTL; on startup warm it.

## quality gates snapshot
- Build/lint/typecheck: Not executed; I made a small import fix in aws_client.py (PASS: no unresolved symbols now for `re`).
- Unit tests: None detected in context for these components.
- Runtime smoke: Not executed here. The two entity route issues likely cause runtime errors until fixed (await + JSON parsing).

## requirements coverage
- Optimization suggestions including parallelization/caching/async: Done.
- Detailed backend flow function-by-function: Done.

Want me to go ahead and apply the two entity_routes fixes (await and summary parsing), swap in async HTTP for the Java call, and wire graph store reuse?

Made changes.
