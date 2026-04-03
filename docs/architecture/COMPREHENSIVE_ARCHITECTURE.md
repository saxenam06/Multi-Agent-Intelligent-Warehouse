# Comprehensive Architecture Document
## Multi-Agent Intelligent Warehouse Operational Assistant (WOSA)

---

# 1. System Overview

The **Warehouse Operational Assistant (WOSA)** is a full-stack, multi-agent AI system for warehouse operations management. It combines a React frontend with a Python FastAPI backend, an NVIDIA NIM-powered LLM engine, a LangGraph multi-agent orchestrator, and a Model Context Protocol (MCP) tool framework — all backed by PostgreSQL/TimescaleDB (relational + time-series), Milvus (vector search), Redis (cache), and Kafka (event streaming).

The system implements a **Retrieval-Augmented Generation (RAG)** pattern: user queries arrive via a chat interface, pass through guardrails, are classified by intent, routed to domain-specific AI agents (Equipment, Operations, Safety, Document, Forecasting), which then execute MCP tools, query databases, and generate LLM-powered responses grounded in retrieved evidence.

The architecture is monorepo-style, with frontend (`src/ui/web/`) and backend (`src/api/`, `src/retrieval/`, `src/adapters/`) co-located. Infrastructure is containerised via Docker Compose with TimescaleDB, Milvus, Redis, Kafka, MinIO, etcd, and Nginx.

---

# 2. Repository Structure

```
Multi-Agent-Intelligent-Warehouse/
├── src/
│   ├── api/                          # FastAPI backend
│   │   ├── app.py                    # Entry point — FastAPI app, middleware, lifespan
│   │   ├── routers/                  # 18 API router modules
│   │   ├── agents/                   # 5 domain AI agents
│   │   │   ├── document/             # Document extraction agent + pipeline
│   │   │   ├── forecasting/          # Demand forecasting agent
│   │   │   ├── inventory/            # Equipment/asset management agent
│   │   │   ├── operations/           # Workforce/task management agent
│   │   │   └── safety/               # Safety incident agent
│   │   ├── graphs/                   # LangGraph orchestration graphs
│   │   ├── services/                 # Business logic layer (20+ services)
│   │   │   ├── auth/                 # JWT auth, user service
│   │   │   ├── cache/                # Redis query cache
│   │   │   ├── deduplication/        # Request deduplication
│   │   │   ├── evidence/             # Evidence collection & integration
│   │   │   ├── guardrails/           # NeMo guardrails + pattern matching
│   │   │   ├── llm/                  # NVIDIA NIM client
│   │   │   ├── mcp/                  # MCP server, tool discovery, routing
│   │   │   │   └── adapters/         # 9 MCP adapters (ERP, WMS, IoT, etc.)
│   │   │   ├── memory/               # Conversation memory & context
│   │   │   ├── monitoring/           # Performance metrics, alerts
│   │   │   ├── quick_actions/        # Smart action recommendations
│   │   │   ├── reasoning/            # Advanced reasoning engine
│   │   │   ├── routing/              # Semantic router
│   │   │   ├── security/             # Rate limiter
│   │   │   └── validation/           # Response quality validation
│   │   ├── middleware/               # Security headers
│   │   └── utils/                    # Error handler, logging
│   ├── retrieval/                    # Retrieval layer
│   │   ├── vector/                   # Milvus retriever, embeddings, chunking
│   │   ├── structured/              # SQL retriever, query specialisations
│   │   ├── caching/                 # Redis caching layer
│   │   └── response_quality/        # Response enhancement, UX analytics
│   ├── adapters/                    # External system adapters
│   │   ├── erp/                     # SAP ECC, Oracle ERP
│   │   ├── wms/                     # SAP EWM, Manhattan, Oracle WMS
│   │   ├── iot/                     # Equipment, environmental, safety sensors
│   │   ├── rfid_barcode/            # Zebra RFID, Honeywell barcode
│   │   └── time_attendance/         # Card, biometric, mobile
│   ├── memory/                      # Memory manager
│   └── ui/web/                      # React frontend
│       ├── src/
│       │   ├── pages/               # 16 page components
│       │   ├── components/          # Shared + chat components
│       │   ├── services/            # API client layer (axios)
│       │   ├── contexts/            # AuthContext
│       │   └── theme/               # NVIDIA dark theme
│       └── craco.config.js          # Build config
├── data/
│   ├── config/                      # Agent YAML configs, guardrails rules
│   └── postgres/                    # SQL schemas + migrations
├── scripts/
│   ├── data/                        # Synthetic data generators
│   ├── setup/                       # Environment & user setup
│   ├── forecasting/                 # Forecasting pipeline scripts
│   └── testing/                     # Test scripts
├── deploy/compose/                  # Docker Compose files (dev, GPU, monitoring)
├── monitoring/                      # Prometheus, Grafana, Alertmanager configs
├── tests/                           # Unit, integration, performance, quality
├── docs/                            # Architecture docs, ADRs, security
├── Dockerfile.backend               # Backend Docker image
├── Dockerfile.frontend              # Frontend Docker image
├── requirements.txt                 # Python dependencies
└── package.json                     # Root (commitlint, husky)
```

---

# 3. Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | React 19, TypeScript, Material-UI 7, TanStack React Query 5, React Router 7, Recharts, Emotion |
| **Backend** | Python 3.12, FastAPI, Uvicorn, Pydantic |
| **AI/LLM** | NVIDIA NIM (`llama-3.3-nemotron-super-49b-v1`), LangGraph, NeMo Guardrails |
| **Embeddings** | NVIDIA `nv-embedqa-e5-v5` (1024-dim) |
| **Relational DB** | PostgreSQL 16 + TimescaleDB 2.15.2 (asyncpg) |
| **Vector DB** | Milvus 2.4.3 (PyMilvus, optional GPU CAGRA index) |
| **Cache** | Redis 7 |
| **Message Queue** | Apache Kafka 3.7 |
| **Object Storage** | MinIO |
| **Reverse Proxy** | Nginx |
| **Observability** | Prometheus, Grafana, Alertmanager |
| **Auth** | JWT (HS256) with access + refresh tokens, bcrypt passwords |
| **CI/CD** | GitHub Actions, SonarQube, Dependabot, Husky + Commitlint |

---

# 4. Backend Architecture

## 4.1 Entry Point

**File:** `src/api/app.py`
**Command:** `python -m uvicorn src.api.app:app --reload --port 8001 --host 0.0.0.0`

The FastAPI app is created at module level with a lifespan context manager that:
- **Startup:** Initialises rate limiter, performance monitor, alert checker
- **Shutdown:** Gracefully closes all services

### Middleware Stack (in order)
1. **Security Headers** — CSP, X-Frame-Options, HSTS
2. **CORS** — Allows `localhost:3000`, `localhost:3001`
3. **Rate Limiter** — Redis-based token bucket per user
4. **Request Size Validation** — 10MB JSON / 50MB upload
5. **Metrics Recording** — Prometheus request metrics

## 4.2 All API Routers & Endpoints

18 routers are mounted under `/api/v1`:

### Chat Router (`/api/v1/chat`) — `src/api/routers/chat.py`

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| POST | `/chat` | `chat()` | Main conversational endpoint — routes to LangGraph agents |
| GET | `/chat/health` | `chat_health()` | Chat subsystem health |
| GET | `/chat/sessions/{session_id}` | `get_session()` | Retrieve conversation session |
| DELETE | `/chat/sessions/{session_id}` | `clear_session()` | Clear conversation history |
| POST | `/chat/feedback` | `submit_feedback()` | User feedback on responses |
| GET | `/chat/stats` | `get_stats()` | Chat performance statistics |
| GET | `/chat/cache/stats` | `get_cache_stats()` | Cache hit/miss statistics |

### Health Router (`/api/v1/health`) — `src/api/routers/health.py`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Full health check (DB, Redis, Milvus, NIM) |
| GET | `/health/simple` | Quick liveness probe |
| GET | `/health/detailed` | Component-level health |
| GET | `/version` | Application version info |
| GET | `/metrics` | Prometheus metrics |
| GET | `/metrics/performance` | Performance summary |

### Equipment Router (`/api/v1/equipment`) — `src/api/routers/equipment.py`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/equipment` | List all equipment assets |
| GET | `/equipment/{asset_id}` | Get asset by ID |
| GET | `/equipment/{asset_id}/status` | Get asset operational status |
| POST | `/equipment/assign` | Assign asset to operator/task |
| POST | `/equipment/release` | Release asset from assignment |
| GET | `/equipment/{asset_id}/telemetry` | Get sensor telemetry (time-range) |
| POST | `/equipment/maintenance/schedule` | Schedule preventive maintenance |
| GET | `/equipment/maintenance/schedule` | List maintenance schedule |
| GET | `/equipment/assignments` | List active assignments |
| GET | `/equipment/maintenance/history` | Maintenance history |

### Operations Router (`/api/v1/operations`) — `src/api/routers/operations.py`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/operations/tasks` | List all tasks |
| POST | `/operations/tasks` | Create new task |
| GET | `/operations/tasks/{task_id}` | Get task by ID |
| POST | `/operations/tasks/{task_id}/assign` | Assign task to worker |
| GET | `/operations/workforce` | Workforce status |
| GET | `/operations/analytics` | Operational analytics |

### Safety Router (`/api/v1/safety`) — `src/api/routers/safety.py`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/safety/incidents` | List safety incidents |
| POST | `/safety/incidents` | Report new incident |
| GET | `/safety/incidents/{id}` | Get incident details |
| GET | `/safety/policies` | Safety policies |
| GET | `/safety/analytics` | Safety analytics |

### Auth Router (`/api/v1/auth`) — `src/api/routers/auth.py`

| Method | Path | Description |
|--------|------|-------------|
| POST | `/auth/login` | Authenticate, return JWT |
| POST | `/auth/refresh` | Refresh access token |
| POST | `/auth/logout` | Revoke refresh token |
| GET | `/auth/me` | Current user profile |
| POST | `/auth/register` | Create new user |
| GET | `/auth/users` | List all users (admin) |
| GET | `/auth/users/public` | List users (for dropdowns) |
| GET | `/auth/users/{id}` | Get user by ID |
| PUT | `/auth/users/{id}` | Update user |
| DELETE | `/auth/users/{id}` | Delete user (admin) |
| PUT | `/auth/users/{id}/password` | Change password |
| PUT | `/auth/users/{id}/status` | Change user status |

### Document Router (`/api/v1/document`) — `src/api/routers/document.py`

| Method | Path | Description |
|--------|------|-------------|
| POST | `/document/upload` | Upload document for extraction |
| GET | `/document/status/{id}` | Processing status |
| GET | `/document/results/{id}` | Extraction results |
| GET | `/document/analytics` | Processing analytics |
| POST | `/document/search` | Search extracted documents |
| POST | `/document/{id}/approve` | Approve extraction |
| POST | `/document/{id}/reject` | Reject extraction |
| GET | `/document/types` | Supported document types |
| GET | `/document/pipeline/status` | Pipeline health |

### WMS Router (`/api/v1/wms`) — `src/api/routers/wms.py`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/wms/connections` | List WMS connections |
| POST | `/wms/connections` | Add WMS connection (SAP EWM, Manhattan, Oracle) |
| DELETE | `/wms/connections/{id}` | Remove connection |
| GET | `/wms/connections/{id}/status` | Connection health |
| GET | `/wms/connections/status` | All connections health |
| GET | `/wms/connections/{id}/inventory` | Get inventory from WMS |
| GET | `/wms/inventory/aggregated` | Aggregated inventory across all WMS |

### IoT Router (`/api/v1/iot`) — `src/api/routers/iot.py`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/iot/connections` | List IoT connections |
| POST | `/iot/connections/{id}` | Add IoT connection |
| DELETE | `/iot/connections/{id}` | Remove connection |
| GET | `/iot/connections/{id}/status` | Connection health |
| GET | `/iot/connections/status` | All connections health |
| GET | `/iot/sensor-readings` | Get sensor data (time-range) |
| GET | `/iot/equipment-status` | Equipment status from sensors |
| GET | `/iot/alerts` | Get alerts (severity, equipment) |
| POST | `/iot/alerts/{id}/acknowledge` | Acknowledge alert |

### ERP Router (`/api/v1/erp`) — `src/api/routers/erp.py`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/erp/connections` | List ERP connections |
| POST | `/erp/connections` | Add ERP connection (SAP, Oracle) |
| DELETE | `/erp/connections/{id}` | Remove connection |
| GET | `/erp/connections/{id}/status` | Connection health |
| GET | `/erp/purchase-orders/{id}` | Get purchase order |
| GET | `/erp/materials/{id}` | Get material master data |
| POST | `/erp/sync` | Sync data between ERP systems |

### Inventory Router (`/api/v1/inventory`) — `src/api/routers/inventory.py`

| Method | Path | Description |
|--------|------|-------------|
| GET | `/inventory/items` | List inventory items |
| GET | `/inventory/items/{sku}` | Get item by SKU |
| POST | `/inventory/items` | Create item |
| PUT | `/inventory/items/{sku}` | Update item |
| GET | `/inventory/movements` | Movement history |
| GET | `/inventory/low-stock` | Low-stock alerts |
| POST | `/inventory/stock-taking` | Record physical count |

### Additional Routers

| Router | Prefix | Purpose |
|--------|--------|---------|
| Scanning | `/api/v1/scanning` | RFID/barcode scan processing |
| Attendance | `/api/v1/attendance` | Employee check-in/out |
| MCP | `/api/v1/mcp` | MCP tool listing & execution |
| Reasoning | `/api/v1/reasoning` | Advanced reasoning chains |
| Forecasting | `/api/v1/forecasting` | Demand forecasting |
| Training | `/api/v1/training` | Model training management |
| Migration | `/api/v1/migration` | Database migrations |

**Total: ~100+ endpoints**

---

## 4.3 Backend Services

All located in `src/api/services/`:

| Service | File(s) | Purpose | Key Methods |
|---------|---------|---------|-------------|
| **NIM Client** | `llm/nim_client.py` | NVIDIA LLM calls | `call_chat()`, `generate_embedding()` |
| **Guardrails** | `guardrails/guardrails_service.py` | Input/output safety | `check_input_safety()`, `check_output_safety()` |
| **NeMo SDK** | `guardrails/nemo_sdk_service.py` | SDK-based guardrails | Colang policy engine |
| **Conversation Memory** | `memory/conversation_memory.py` | Chat history storage | `add_memory()`, `retrieve_memory()` |
| **Context Enhancer** | `memory/context_enhancer.py` | Response enrichment | `enhance_with_context()` |
| **Query Cache** | `cache/query_cache.py` | Redis response cache | `get()`, `set()` (5-min TTL) |
| **Request Deduplicator** | `deduplication/request_deduplicator.py` | Duplicate prevention | `get_or_create_task()` |
| **Rate Limiter** | `security/rate_limiter.py` | Token bucket rate limiting | `check_rate_limit()` |
| **JWT Handler** | `auth/jwt_handler.py` | Token generation/verification | `create_token_pair()`, `verify_token()` |
| **User Service** | `auth/user_service.py` | User CRUD | `create_user()`, `authenticate_user()` |
| **Evidence Integration** | `evidence/evidence_integration.py` | Evidence gathering | `enhance_response_with_evidence()` |
| **Smart Quick Actions** | `quick_actions/smart_quick_actions.py` | Action suggestions | `generate_quick_actions()` |
| **Performance Monitor** | `monitoring/performance_monitor.py` | Latency tracking | `start_request()`, `end_request()` |
| **Alert Checker** | `monitoring/alert_checker.py` | Threshold alerting | `start()`, `check()` |
| **Response Validator** | `validation/response_validator.py` | Response quality | `validate()`, `score_response()` |
| **Reasoning Engine** | `reasoning/reasoning_engine.py` | Multi-step reasoning | Chain-of-thought, tree-of-thought |
| **Semantic Router** | `routing/semantic_router.py` | Query classification | Intent-based routing |
| **MCP Tool Discovery** | `mcp/tool_discovery.py` | Dynamic tool finding | `discover_all_tools()`, `search_tools()` |
| **MCP Tool Binding** | `mcp/tool_binding.py` | Tool-to-agent binding | `bind_tools()`, `create_execution_plan()` |
| **MCP Tool Routing** | `mcp/tool_routing.py` | Tool selection strategy | Cost/latency/reliability optimised |

---

# 5. LangGraph Agent Orchestration

## 5.1 Graph Architecture

**File:** `src/api/graphs/mcp_integrated_planner_graph.py` (production graph)

The system has three generations of planner graphs:
1. `planner_graph.py` — Base implementation (no MCP)
2. `mcp_planner_graph.py` — Phase 2 (basic MCP)
3. `mcp_integrated_planner_graph.py` — Phase 3 (full MCP integration, in use)

### State Schema

```python
class MCPWarehouseState(TypedDict):
    messages: List[BaseMessage]          # Chat messages
    user_intent: Optional[str]           # Classified intent
    routing_decision: Optional[str]      # Agent to route to
    agent_responses: Dict[str, str]      # Agent outputs
    final_response: Optional[str]        # Synthesised response
    context: Dict[str, Any]              # Session context
    session_id: str                      # Session ID
    mcp_results: Optional[Any]           # MCP tool results
    tool_execution_plan: Optional[List]  # Planned tool calls
    available_tools: Optional[List]      # Discovered tools
    enable_reasoning: bool               # Advanced reasoning flag
    reasoning_types: Optional[List[str]] # Reasoning methods
    reasoning_chain: Optional[Dict]      # Reasoning trace
```

### Graph Nodes & Flow

```
              ┌──────────────┐
              │ route_intent  │  ← MCPIntentClassifier
              └──────┬───────┘
                     │
     ┌───────┬───────┼───────┬──────────┬──────────┬──────────┐
     ▼       ▼       ▼       ▼          ▼          ▼          ▼
equipment operations safety forecasting document  general  ambiguous
     │       │       │       │          │          │          │
     └───────┴───────┴───────┴──────────┴──────────┴──────────┘
                                │
                         ┌──────▼──────┐
                         │  synthesize  │
                         └──────┬──────┘
                                │
                               END
```

### Node Detail

| Node | Function | LLM Call? | Description |
|------|----------|-----------|-------------|
| `route_intent` | `_mcp_route_intent()` | No | Keyword + semantic intent classification |
| `equipment` | `_mcp_equipment_agent()` | Yes | Equipment status, assignment, telemetry, maintenance |
| `operations` | `_mcp_operations_agent()` | Yes | Workforce, tasks, scheduling, pick waves |
| `safety` | `_mcp_safety_agent()` | Yes | Incidents, compliance, LOTO, hazards |
| `forecasting` | `_mcp_forecasting_agent()` | Yes | Demand prediction, reorder recommendations |
| `document` | `_mcp_document_agent()` | Yes | OCR, extraction, validation, routing |
| `general` | `_mcp_general_agent()` | Yes | Fallback for unclassified queries |
| `ambiguous` | `_handle_ambiguous_query()` | No | Generates clarifying questions |
| `synthesize` | `_mcp_synthesize_response()` | No | Formats final response with metadata |

### Intent Classification Priority

The `MCPIntentClassifier.classify_intent()` method uses keyword matching with this priority:

1. **Worker keywords** → `operations` (override: "worker", "workforce", "employee", "staff")
2. **Forecasting** → `forecasting` ("forecast", "predict", "demand planning", "reorder")
3. **Safety** → `safety` (emergency keywords OR safety context score ≥ 2)
4. **Document** → `document` ("invoice", "PDF", "OCR", "extract")
5. **Equipment** → `equipment` (equipment indicators + equipment objects, excluding workflows)
6. **Operations** → `operations` ("wave", "order", "pick", "pack", "create")
7. **Fallback** → `equipment`

### Timeout Configuration

| Query Type | Timeout |
|-----------|---------|
| Simple | 60s |
| Complex (>15 words or complex keywords) | 90s |
| With reasoning | 115s |
| Complex + reasoning | 230s |

---

## 5.2 AI Agents

### Equipment Agent — `src/api/agents/inventory/mcp_equipment_agent.py`

- **Tools:** Asset CRUD, telemetry query, assignment, maintenance scheduling
- **Data sources:** `equipment_assets`, `equipment_telemetry`, `equipment_assignments`, `equipment_maintenance`
- **Action tools:** `equipment_action_tools.py`, `equipment_asset_tools.py`

### Operations Agent — `src/api/agents/operations/mcp_operations_agent.py`

- **Tools:** Task creation, assignment, workforce scheduling, pick wave creation
- **Data sources:** `tasks`, `users`
- **Action tools:** `action_tools.py`

### Safety Agent — `src/api/agents/safety/mcp_safety_agent.py`

- **Tools:** Incident reporting, policy retrieval, alert acknowledgement
- **Data sources:** `safety_incidents`
- **Action tools:** `action_tools.py`

### Forecasting Agent — `src/api/agents/forecasting/forecasting_agent.py`

- **Tools:** Demand forecasting, model training, reorder recommendations
- **Data sources:** `inventory_movements`, `inventory_items`
- **Action tools:** `forecasting_action_tools.py`

### Document Agent — `src/api/agents/document/mcp_document_agent.py`

Multi-stage pipeline:

```
Upload → Preprocessing → OCR (Nemotron) → Small LLM Extraction
  → Embedding + Indexing (Milvus) → Quality Validation (Large LLM Judge)
    → Intelligent Routing → WMS Integration
```

**Sub-modules:**
- `preprocessing/layout_detection.py` — Document layout analysis
- `ocr/nemo_ocr.py`, `ocr/nemotron_parse.py` — NVIDIA OCR integration
- `processing/small_llm_processor.py` — Structured field extraction
- `processing/embedding_indexing.py` — Milvus vector storage
- `validation/quality_scorer.py` — Accuracy scoring
- `validation/large_llm_judge.py` — LLM-based quality review
- `routing/intelligent_router.py` — Action routing

---

# 6. MCP (Model Context Protocol) Framework

**Location:** `src/api/services/mcp/`

## 6.1 Architecture

```
┌─────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Tool Discovery  │────▶│   Tool Binding    │────▶│  Tool Execution  │
│  (dynamic scan)  │     │  (strategy-based) │     │  (with fallback) │
└─────────────────┘     └──────────────────┘     └──────────────────┘
         │                        │                        │
         ▼                        ▼                        ▼
┌─────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Tool Validation │     │   Tool Routing    │     │    Monitoring     │
│  (security)      │     │  (optimisation)   │     │  (performance)   │
└─────────────────┘     └──────────────────┘     └──────────────────┘
```

## 6.2 MCP Adapters

9 adapters in `src/api/services/mcp/adapters/`:

| Adapter | File | External System |
|---------|------|----------------|
| Equipment | `equipment_adapter.py` | Equipment telemetry & management |
| Operations | `operations_adapter.py` | Task & workforce management |
| Safety | `safety_adapter.py` | Safety incident reporting |
| Forecasting | `forecasting_adapter.py` | Demand forecasting |
| ERP | `erp_adapter.py` | SAP ECC, Oracle ERP |
| WMS | `wms_adapter.py` | Manhattan, JDA WMS |
| IoT | `iot_adapter.py` | Sensor & device data |
| RFID/Barcode | `rfid_barcode_adapter.py` | Scanning systems |
| Time & Attendance | `time_attendance_adapter.py` | Employee attendance |

## 6.3 Tool Binding Strategies

| Strategy | Description |
|----------|-------------|
| EXACT_MATCH | Tool name matches exactly |
| FUZZY_MATCH | Fuzzy string matching |
| SEMANTIC_MATCH | Embedding-based similarity |
| CATEGORY_MATCH | Category-based matching |
| PERFORMANCE_BASED | By success rate & latency |

## 6.4 Execution Modes

`SEQUENTIAL`, `PARALLEL`, `PIPELINE`, `CONDITIONAL`

---

# 7. Data Layer

## 7.1 PostgreSQL / TimescaleDB

**Container:** `timescaledb:2.15.2-pg16` — Port 5435
**Driver:** asyncpg (connection pool: min 1, max 10)
**ORM:** None — raw parameterised SQL via `SQLRetriever` singleton

### Tables

| Table | Schema File | Purpose | Type |
|-------|------------|---------|------|
| `inventory_items` | `000_schema.sql` | SKU master data (Frito-Lay products) | Relational |
| `tasks` | `000_schema.sql` | Warehouse tasks (pick, pack, putaway) | Relational |
| `safety_incidents` | `000_schema.sql` | Safety incident records | Relational |
| `users` | `000_schema.sql` | User authentication & roles | Relational |
| `user_sessions` | `000_schema.sql` | JWT refresh token sessions | Relational |
| `audit_log` | `000_schema.sql` | User action audit trail | Relational |
| `equipment_assets` | `001_equipment_schema.sql` | Equipment master (forklift, AMR, scanner) | Relational |
| `equipment_assignments` | `001_equipment_schema.sql` | Equipment-to-operator assignments | Relational |
| `equipment_telemetry` | `001_equipment_schema.sql` | Sensor readings (battery, temp, speed) | **Hypertable** |
| `equipment_maintenance` | `001_equipment_schema.sql` | Maintenance records | Relational |
| `equipment_performance` | `001_equipment_schema.sql` | Performance metrics | Relational |
| `documents` | `002_document_schema.sql` | Uploaded documents metadata | Relational |
| `document_operations` | `002_document_schema.sql` | Document-to-operation links | Relational |
| `processing_stages` | `002_document_schema.sql` | Pipeline stage tracking | Relational |
| `extraction_results` | `002_document_schema.sql` | OCR/LLM extraction data | Relational |
| `quality_scores` | `002_document_schema.sql` | Quality validation scores | Relational |
| `routing_decisions` | `002_document_schema.sql` | Intelligent routing decisions | Relational |
| `document_search_metadata` | `002_document_schema.sql` | Vector search metadata | Relational |
| `inventory_movements` | `004_inventory_movements_schema.sql` | Stock movement history | Relational |

### Views

| View | Purpose |
|------|---------|
| `daily_demand` | Daily outbound demand by SKU |
| `weekly_demand` | Weekly outbound demand by SKU |
| `monthly_demand` | Monthly outbound demand by SKU |
| `brand_demand` | Monthly demand by brand (SKU prefix) |

### Query Specialisations

| Module | File | Tables |
|--------|------|--------|
| Task Queries | `src/retrieval/structured/task_queries.py` | tasks |
| Inventory Queries | `src/retrieval/structured/inventory_queries.py` | inventory_items, inventory_movements |
| Telemetry Queries | `src/retrieval/structured/telemetry_queries.py` | equipment_telemetry |
| SQL Query Router | `src/retrieval/structured/sql_query_router.py` | Routes to specialised handlers |

## 7.2 Milvus (Vector Database)

**Container:** `milvusdb/milvus:v2.4.3` — Port 19530
**Dependencies:** etcd (metadata), MinIO (storage)

### Collections

| Collection | Dimension | Index | Purpose |
|-----------|-----------|-------|---------|
| `warehouse_docs` | 1024 | IVF_FLAT (CPU) or GPU_CAGRA (GPU) | Document embeddings |
| `warehouse_docs_gpu` | 1024 | GPU_CAGRA | GPU-accelerated variant |

### Schema

```
Fields:
  id          VARCHAR(100)    — primary key
  content     VARCHAR(65535)  — document text
  embedding   FLOAT_VECTOR    — 1024-dim (nv-embedqa-e5-v5)
  doc_type    VARCHAR(50)     — document category
  category    VARCHAR(100)    — sub-category
  created_at  VARCHAR(50)     — timestamp
```

### Retrieval Chain

```
Query → EmbeddingService.generate_embedding() → MilvusRetriever.search()
  → HybridRanker.rerank() → EvidenceScoring.score() → Results
```

## 7.3 Redis

**Container:** `redis:7` — Port 6379

| Key Pattern | TTL | Purpose |
|------------|-----|---------|
| `query_cache:{hash}:{session}` | 300s | Query response cache |
| `rate_limit:{user_id}` | Rolling | Token bucket state |
| `session:{session_id}` | Variable | Session data |

## 7.4 Kafka

**Container:** `apache/kafka:3.7.0` — Port 9092

Used for document processing job queue, async task distribution, and event streaming.

## 7.5 MinIO

**Container:** `minio/minio` — Ports 9000/9001

Milvus vector data storage and document file storage.

---

# 8. Retrieval Architecture

## 8.1 Hybrid Retrieval

```
User Query
    │
    ├──▶ EmbeddingService (NVIDIA nv-embedqa-e5-v5) ──▶ MilvusRetriever (semantic)
    │
    ├──▶ QueryPreprocessing ──▶ SQLRetriever (structured)
    │
    └──▶ Full-text search (PostgreSQL FTS) ──▶ (lexical)
    │
    ▼
HybridRanker (RRF / BM25 / L2R)
    │
    ▼
ResultPostprocessing
    │
    ▼
Evidence Scoring & Ranking
```

### Key Classes

| Class | File | Purpose |
|-------|------|---------|
| `EmbeddingService` | `src/retrieval/vector/embedding_service.py` | Text → 1024-dim vectors |
| `MilvusRetriever` | `src/retrieval/vector/milvus_retriever.py` | Vector similarity search |
| `GPUMilvusRetriever` | `src/retrieval/vector/gpu_milvus_retriever.py` | GPU-accelerated search |
| `SQLRetriever` | `src/retrieval/structured/sql_retriever.py` | Parameterised SQL queries |
| `HybridRetriever` | `src/retrieval/hybrid_retriever.py` | Combined vector + SQL |
| `EnhancedHybridRetriever` | `src/retrieval/enhanced_hybrid_retriever.py` | With re-ranking |
| `ChunkingService` | `src/retrieval/vector/chunking_service.py` | Document chunking |
| `ClarifyingQuestions` | `src/retrieval/vector/clarifying_questions.py` | Follow-up generation |

---

# 9. Guardrails

**Location:** `src/api/services/guardrails/`

### Input Guardrails (applied to user message)

- Prompt injection detection
- PII detection (phone, email, SSN patterns)
- Toxic language detection
- SQL injection pattern matching
- Malicious intent detection

### Output Guardrails (applied to LLM response)

- Hallucination detection
- Consistency checks
- Harmful content filtering
- PII leakage prevention
- Factual accuracy validation

### Implementation Modes

1. **NeMo SDK** — Colang policy engine (`nemo_sdk_service.py`, requires `USE_NEMO_GUARDRAILS_SDK=true`)
2. **Pattern Matching** — YAML + regex based (`guardrails_service.py`, default)

---

# 10. Frontend Architecture

## 10.1 Tech Stack

| Aspect | Technology |
|--------|-----------|
| Framework | React 19.2.3 + TypeScript |
| Build | CRACO (wraps react-scripts) |
| UI Library | MUI 7 |
| State | TanStack React Query 5 (server), React Context (auth), useState (UI) |
| Routing | React Router 7 (BrowserRouter) |
| API Client | Axios with interceptors |
| Charts | Recharts |
| Theme | NVIDIA dark theme (`#76B900` green, `#00D9FF` cyan, `#0A0E13` bg) |

## 10.2 Pages & Routing

```
/login              → Login (public)
/                   → Dashboard (protected — ProtectedRoute wraps all below)
/chat               → ChatInterfaceNew
/equipment          → EquipmentNew
/forecasting        → Forecasting
/operations         → Operations
/safety             → Safety
/documents          → DocumentExtraction
/analytics          → Analytics
/documentation      → Documentation
/documentation/mcp-integration    → MCPIntegrationGuide
/documentation/api-reference      → APIReference
/documentation/deployment         → DeploymentGuide
/documentation/architecture       → ArchitectureDiagrams
/mcp-test           → MCPTest
```

## 10.3 Component Hierarchy

```
App
├── Login (public)
└── ProtectedRoute → Layout
    ├── AppBar (logo, user menu, logout)
    ├── Sidebar (10 navigation items)
    └── Routes
        ├── Dashboard
        ├── ChatInterfaceNew
        │   ├── TopBar (warehouse, role, env, connections)
        │   ├── LeftRail (quick actions, demo scripts)
        │   ├── Chat Area
        │   │   ├── MessageBubble[] (per message)
        │   │   └── Input (message field, reasoning toggle, send)
        │   └── RightPanel (evidence, reasoning chain, SQL, context)
        ├── EquipmentNew (DataGrid, tabs, dialogs)
        ├── Forecasting (dashboard, training, recommendations)
        ├── Operations (tasks DataGrid, workforce, assignment)
        ├── Safety (incidents DataGrid, policies, reporting)
        ├── DocumentExtraction (upload, results, approval)
        ├── Analytics (charts: equipment, incidents)
        └── Documentation pages
```

## 10.4 API Service Layer

**File:** `src/ui/web/src/services/api.ts`

**Base URL:** `/api/v1` (proxied to `http://localhost:8001` via `setupProxy.js`)

**Auth:** Axios request interceptor adds `Authorization: Bearer {token}` from localStorage.

**All API modules:**

| Module | Key Methods | Backend Endpoints |
|--------|------------|-------------------|
| `chatAPI` | `sendMessage()` | `POST /chat` |
| `equipmentAPI` | `getAllAssets()`, `assignAsset()`, `getTelemetry()`, `scheduleMaintenance()` | `/equipment/*` |
| `operationsAPI` | `getTasks()`, `assignTask()`, `getWorkforceStatus()` | `/operations/*` |
| `safetyAPI` | `getIncidents()`, `reportIncident()`, `getPolicies()` | `/safety/*` |
| `documentAPI` | `uploadDocument()`, `getDocumentResults()`, `approveDocument()` | `/document/*` |
| `inventoryAPI` | `getAllItems()`, `createItem()`, `updateItem()` | `/inventory/*` |
| `healthAPI` | `check()` | `GET /health/simple` |
| `userAPI` | `getUsers()` | `GET /auth/users/public` |
| `mcpAPI` | `getStatus()`, `getTools()`, `executeTool()` | `/mcp/*` |
| `trainingAPI` | `startTraining()`, `getTrainingStatus()`, `stopTraining()` | `/training/*` |
| `forecastingAPI` | `getDashboardSummary()`, `getRealTimeForecast()`, `getReorderRecommendations()` | `/forecasting/*` |

## 10.5 Authentication Flow

```
1. User enters credentials on /login
2. POST /api/v1/auth/login → JWT token
3. Token stored in localStorage (auth_token)
4. JWT decoded → user info stored (user_info)
5. Axios interceptor adds Bearer token to all requests
6. On 401 → token removed, redirect to /login
7. On app load → GET /api/v1/auth/me verifies token
```

---

# 11. End-to-End Traces

## 11.1 Chat Message: "What forklifts are available in Zone A?"

```
┌─ FRONTEND ──────────────────────────────────────────────────────────┐
│ ChatInterfaceNew                                                     │
│   └─ sendMessageMutation.mutateAsync({                              │
│        message: "What forklifts are available in Zone A?",           │
│        session_id: "default",                                        │
│        context: { warehouse: "WH-01", role: "operator" },           │
│        enable_reasoning: false                                       │
│      })                                                              │
│   └─ chatAPI.sendMessage()                                          │
│      └─ axios.post('/api/v1/chat', payload, { timeout: 60000 })    │
│         └─ Proxy (setupProxy.js) → http://localhost:8001/api/v1/chat│
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─ BACKEND ────────────────────────────────────────────────────────────┐
│ 1. MIDDLEWARE CHAIN                                                   │
│    Security Headers → CORS → Rate Limiter → Size Validation → Metrics│
│                                                                       │
│ 2. CHAT ROUTER (src/api/routers/chat.py:598)                        │
│    async def chat(req: ChatRequest)                                   │
│    ├─ check_rate_limit()                                             │
│    ├─ guardrails_service.check_input_safety(message)                 │
│    │   └─ Pattern matching: no violations → is_safe=True             │
│    ├─ query_cache.get(message, session_id)                           │
│    │   └─ Cache miss                                                 │
│    │                                                                  │
│ 3. LANGGRAPH INVOCATION                                              │
│    ├─ get_mcp_planner_graph()                                        │
│    │   └─ Returns compiled StateGraph                                │
│    ├─ graph.ainvoke(initial_state)                                   │
│    │                                                                  │
│    ├─ NODE: route_intent                                             │
│    │   └─ MCPIntentClassifier.classify_intent(message)               │
│    │   └─ "forklift" matches EQUIPMENT_KEYWORDS                      │
│    │   └─ routing_decision = "equipment"                             │
│    │                                                                  │
│    ├─ NODE: equipment                                                │
│    │   └─ MCPEquipmentAgent.process_query(message, context)          │
│    │   ├─ MCP tool binding: equipment_adapter → get_assets tool      │
│    │   ├─ SQL query via SQLRetriever:                                │
│    │   │   SELECT * FROM equipment_assets                            │
│    │   │   WHERE type='forklift' AND zone='Zone A'                   │
│    │   │     AND status='available'                                   │
│    │   ├─ NIM LLM call (nemotron-super-49b):                        │
│    │   │   System: "You are an equipment management assistant..."    │
│    │   │   User: query + SQL results as context                      │
│    │   │   → natural_language: "There is 1 forklift available..."    │
│    │   └─ Returns: { natural_language, structured_data, confidence } │
│    │                                                                  │
│    ├─ NODE: synthesize                                               │
│    │   └─ Extracts natural_language, structured_response              │
│    │   └─ Adds mcp_tools_used, tool_execution_results                │
│    │                                                                  │
│ 4. POST-PROCESSING                                                   │
│    ├─ _format_user_response(graph_result)                            │
│    ├─ evidence_integration.enhance_response_with_evidence()          │
│    ├─ context_enhancer.enhance_with_context()                        │
│    ├─ smart_quick_actions.generate_quick_actions()                   │
│    ├─ guardrails_service.check_output_safety(response)               │
│    ├─ query_cache.set(message, response)  → cache for 5 min         │
│    └─ Return ChatResponse                                            │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─ FRONTEND ──────────────────────────────────────────────────────────┐
│ chatMutation.onSuccess(response)                                     │
│   └─ simulateStreamingResponse(response)                            │
│      ├─ Add message to messages[] state                              │
│      ├─ Generate streaming events (UI animation)                     │
│      ├─ Extract evidence → currentEvidence state                    │
│      ├─ Extract reasoning_chain → currentReasoningChain state       │
│      └─ Auto-scroll to latest message                               │
│                                                                       │
│ MessageBubble renders:                                               │
│   ├─ Agent icon (equipment = green)                                  │
│   ├─ Confidence score badge                                          │
│   ├─ Response text                                                   │
│   └─ Structured data table (if available)                           │
│                                                                       │
│ RightPanel shows:                                                    │
│   ├─ Evidence accordion (SQL query, results)                        │
│   ├─ Planner decision ("equipment" route)                           │
│   └─ Tool timeline                                                   │
└──────────────────────────────────────────────────────────────────────┘
```

## 11.2 Equipment Assignment

```
Frontend: EquipmentNew
  └─ User clicks "Assign" → opens dialog
  └─ Fills: asset_id=FL-01, assignee=operator1, type=task, task_id=TASK-005
  └─ assignMutation.mutateAsync(data)
    └─ equipmentAPI.assignAsset(data)
      └─ axios.post('/api/v1/equipment/assign', data)

Backend: src/api/routers/equipment.py
  └─ POST /equipment/assign
    └─ Validate asset exists (SELECT FROM equipment_assets WHERE asset_id=$1)
    └─ INSERT INTO equipment_assignments (asset_id, task_id, assignee, assignment_type)
    └─ UPDATE equipment_assets SET status='assigned', owner_user=$1 WHERE asset_id=$2
    └─ Return assignment record

Frontend:
  └─ onSuccess: invalidate ['equipment-assignments', 'equipment']
  └─ React Query refetches → DataGrid updates
  └─ Close dialog, show success snackbar
```

## 11.3 Safety Incident Report

```
Frontend: Safety page
  └─ User clicks "Report Incident"
  └─ Fills: severity=high, description="Forklift near-miss in Zone B", reported_by=supervisor1
  └─ reportMutation.mutateAsync(data)
    └─ safetyAPI.reportIncident(data)
      └─ axios.post('/api/v1/safety/incidents', data)

Backend: src/api/routers/safety.py
  └─ POST /safety/incidents
    └─ INSERT INTO safety_incidents (severity, description, reported_by, occurred_at)
    └─ Return incident record with ID

Frontend:
  └─ onSuccess: invalidate ['incidents']
  └─ React Query refetches → DataGrid shows new incident
```

## 11.4 Document Upload & Processing

```
Frontend: DocumentExtraction page
  └─ User drops file on upload zone
  └─ uploadMutation.mutateAsync(formData)
    └─ documentAPI.uploadDocument(formData)
      └─ axios.post('/api/v1/document/upload', formData, {multipart})

Backend: src/api/routers/document.py
  └─ POST /document/upload
    └─ Save file → INSERT INTO documents (filename, file_path, file_type, status='uploaded')
    └─ Trigger pipeline:
      1. preprocessing/layout_detection.py → layout analysis
      2. ocr/nemotron_parse.py → NVIDIA Nemotron vision → text
         └─ INSERT INTO processing_stages (stage='ocr', status='completed')
         └─ INSERT INTO extraction_results (stage='ocr', raw_data=...)
      3. processing/small_llm_processor.py → NIM call → structured fields
         └─ INSERT INTO extraction_results (stage='llm', processed_data=...)
      4. processing/embedding_indexing.py → generate embedding → Milvus insert
         └─ INSERT INTO document_search_metadata (search_vector_id, embedding_model)
      5. validation/large_llm_judge.py → NIM call → quality scores
         └─ INSERT INTO quality_scores (overall_score, decision)
      6. routing/intelligent_router.py → route decision
         └─ INSERT INTO routing_decisions (routing_action, wms_integration_status)
    └─ UPDATE documents SET status='completed'
    └─ Return document ID + processing summary

Frontend:
  └─ Poll GET /document/status/{id} for progress
  └─ When complete: GET /document/results/{id}
  └─ Display extracted data, confidence scores
  └─ User can approve/reject → POST /document/{id}/approve
```

## 11.5 Dashboard Load

```
Frontend: Dashboard
  └─ 4 parallel React Query fetches on mount:
    ├─ useQuery(['health']) → GET /api/v1/health/simple
    ├─ useQuery(['equipment']) → GET /api/v1/equipment
    ├─ useQuery(['tasks']) → GET /api/v1/operations/tasks
    └─ useQuery(['incidents']) → GET /api/v1/safety/incidents

Backend: Each endpoint queries PostgreSQL via SQLRetriever
  ├─ health → SELECT 1 (DB ping) + Redis ping + Milvus ping
  ├─ equipment → SELECT * FROM equipment_assets
  ├─ tasks → SELECT * FROM tasks ORDER BY created_at DESC
  └─ incidents → SELECT * FROM safety_incidents ORDER BY occurred_at DESC

Frontend: Computes derived data from query results
  ├─ maintenanceNeeded = assets where status='maintenance' OR next_pm_due <= now
  ├─ pendingTasks = tasks where status='pending'
  ├─ recentIncidents = incidents.slice(0, 5)
  └─ Renders 4 stat cards + 3 detail lists
```

## 11.6 Forecasting Model Training

```
Frontend: Forecasting page → Model Training tab
  └─ User selects training_type='advanced', clicks Start
  └─ useMutation → trainingAPI.startTraining({ training_type: 'advanced' })
    └─ POST /api/v1/training/start

Backend: src/api/routers/training.py
  └─ Triggers background training job
  └─ Returns { job_id, status: 'started' }

Frontend: Polls every 2 seconds
  └─ useQuery(['training-status'], refetchInterval: 2000)
    └─ trainingAPI.getTrainingStatus()
      └─ GET /api/v1/training/status
      └─ Returns { is_running, progress: 45, current_step: 'feature_engineering', logs }

  └─ UI shows: progress bar, current step name, streaming logs
  └─ When progress = 100 & is_running = false:
    └─ invalidate ['training-history', 'forecasting-dashboard']
    └─ Show completion notification
```

## 11.7 Chat with Advanced Reasoning

```
Frontend: ChatInterfaceNew
  └─ User enables reasoning toggle
  └─ Selects: chain_of_thought, multi_hop
  └─ Sends: "Analyze the relationship between equipment downtime and order delays"
  └─ chatAPI.sendMessage({
       message: ...,
       enable_reasoning: true,
       reasoning_types: ['chain_of_thought', 'multi_hop']
     })
  └─ Dynamic timeout: 240s (complex + reasoning)

Backend:
  └─ route_intent → detects "analyze" + "relationship" → complex query
  └─ Routes to operations agent (workflow terms)
  └─ Agent creates reasoning chain:
      Step 1: Query equipment_telemetry for downtime events
      Step 2: Query tasks for delayed orders (status='pending' past deadline)
      Step 3: Cross-reference timestamps
      Step 4: LLM synthesis with evidence
  └─ NIM call with reasoning context → structured analysis
  └─ Returns response + reasoning_chain object

Frontend:
  └─ RightPanel → ReasoningChainVisualization component
    └─ Shows step-by-step reasoning trace
    └─ Evidence links for each step
    └─ Confidence scores per reasoning hop
```

---

# 12. Ingestion / Offline Pipelines

**Location:** `scripts/data/`

| Script | Data | Processing | Target | Trigger |
|--------|------|-----------|--------|---------|
| `generate_synthetic_data.py` | Inventory, tasks, equipment, incidents, attendance, telemetry | Random generation with realistic patterns | PostgreSQL (all core tables) | Manual / `run_data_generation.sh` |
| `generate_equipment_telemetry.py` | 7-day equipment sensor data | Hourly samples per metric per asset | `equipment_telemetry` hypertable | Manual |
| `generate_historical_demand.py` | 90-day inventory movements | Daily outbound quantities with seasonality | `inventory_movements` | Manual |
| `generate_all_sku_forecasts.py` | Forecast outputs | Runs forecasting models | JSON files in `data/sample/forecasts/` | Manual |
| `quick_demo_data.py` | Minimal demo dataset | Subset of synthetic data | PostgreSQL | Manual / `run_quick_demo.sh` |
| `create_default_users.py` | Default users | Hashes passwords from env vars | `users` table | Manual |

### Forecasting Pipeline Scripts (`scripts/forecasting/`)

| Script | Purpose |
|--------|---------|
| `phase1_phase2_forecasting_agent.py` | Basic + intermediate demand forecasting |
| `phase3_advanced_forecasting.py` | Advanced multi-model forecasting |
| `rapids_gpu_forecasting.py` | RAPIDS GPU-accelerated forecasting |

### Full Offline → Online Data Flow

```
1. SCHEMA INIT
   data/postgres/000-004_*.sql → Docker init → PostgreSQL tables + hypertables

2. DATA GENERATION
   scripts/data/generate_synthetic_data.py
     → inventory_items (16 Frito-Lay SKUs)
     → tasks (pick, pack, putaway)
     → equipment_assets (12 assets: forklifts, AMR, AGV, scanners)
     → safety_incidents (various severities)
     → equipment_telemetry (7 days hourly)
     → inventory_movements (90 days)

3. USER SETUP
   scripts/setup/create_default_users.py
     → users (admin, supervisor, operator, viewer)

4. DOCUMENT INGESTION (runtime)
   POST /document/upload → Document pipeline
     → OCR → LLM extraction → Embeddings → Milvus
     → Quality validation → PostgreSQL metadata

5. ONLINE RETRIEVAL
   Chat query → LangGraph agent → SQLRetriever (PostgreSQL) + MilvusRetriever (vectors)
     → Hybrid ranking → Evidence scoring → LLM synthesis → Response
```

---

# 13. Infrastructure & Deployment

## 13.1 Docker Compose Services

**File:** `deploy/compose/docker-compose.dev.yaml`

| Service | Image | Port | Purpose |
|---------|-------|------|---------|
| `wosa-backend` | `Dockerfile.backend` | 8001 | FastAPI backend |
| `wosa-frontend` | `Dockerfile.frontend` | 3001 | React frontend |
| `wosa-nginx` | `nginx:alpine` | 3000 | Reverse proxy |
| `timescaledb` | `timescaledb:2.15.2-pg16` | 5435 | Relational + time-series DB |
| `redis` | `redis:7` | 6379 | Cache + rate limiting |
| `milvus` | `milvusdb/milvus:v2.4.3` | 19530 | Vector database |
| `etcd` | `quay.io/coreos/etcd:v3.5.9` | 2379 | Milvus metadata |
| `minio` | `minio/minio` | 9000/9001 | Object storage |
| `kafka` | `apache/kafka:3.7.0` | 9092 | Event streaming |

## 13.2 Monitoring Stack

**File:** `deploy/compose/docker-compose.monitoring.yaml`

- **Prometheus** — Scrapes `/api/v1/metrics`
- **Grafana** — 3 dashboards: overview, operations, safety
- **Alertmanager** — Threshold-based alerting

---

# 14. Key Design Patterns

| Pattern | Where | How |
|---------|-------|-----|
| **RAG** | Chat pipeline | Query → retrieve evidence (vector + SQL) → LLM synthesis |
| **Multi-Agent Orchestration** | LangGraph | Intent classification → route to domain agent |
| **MCP Tool Framework** | `services/mcp/` | Dynamic tool discovery, binding, execution with fallback |
| **Singleton** | `SQLRetriever` | Single connection pool shared across requests |
| **Adapter** | `src/adapters/` | Abstract base → vendor implementations (SAP, Oracle, etc.) |
| **Factory** | `adapters/*/factory.py` | Create adapter instances by type |
| **Token Bucket** | Rate limiter | Redis-backed per-user rate limiting |
| **Circuit Breaker** | NIM client | Timeout + retry with exponential backoff |
| **Cache-Aside** | Query cache | Check Redis → miss → compute → store |
| **Request Deduplication** | Deduplicator service | Same query + session → share single computation |
| **CQRS-lite** | Retrieval layer | Separate read (hybrid retriever) and write (ingestion) paths |
| **Guardrails** | Input + output | Safety checks on both user input and LLM output |

---

# 15. Dependency Map

## Backend Module Dependencies

```
app.py
├── routers/ (18 modules)
│   └── chat.py
│       ├── services/guardrails/guardrails_service
│       ├── services/cache/query_cache
│       ├── services/deduplication/request_deduplicator
│       ├── services/evidence/evidence_integration
│       ├── services/memory/context_enhancer
│       ├── services/quick_actions/smart_quick_actions
│       ├── services/validation/response_validator
│       └── graphs/mcp_integrated_planner_graph
│           ├── agents/inventory/mcp_equipment_agent
│           ├── agents/operations/mcp_operations_agent
│           ├── agents/safety/mcp_safety_agent
│           ├── agents/forecasting/forecasting_agent
│           ├── agents/document/mcp_document_agent
│           └── services/mcp/tool_discovery + tool_binding
│               └── services/mcp/adapters/* (9 adapters)
│                   └── adapters/* (external system adapters)
├── services/llm/nim_client (used by all agents)
├── retrieval/vector/embedding_service (used by agents + document pipeline)
├── retrieval/vector/milvus_retriever (used by evidence, document)
├── retrieval/structured/sql_retriever (used by all agents)
└── services/auth/* (used by auth router + middleware)
```

## Frontend Component Dependencies

```
App.tsx
├── AuthContext (contexts/AuthContext.tsx)
├── Layout (components/Layout.tsx)
│   └── Sidebar navigation → React Router
├── ChatInterfaceNew
│   ├── chatAPI.sendMessage → POST /chat
│   ├── TopBar, LeftRail, MessageBubble, RightPanel
│   └── ReasoningChainVisualization
├── EquipmentNew
│   ├── equipmentAPI.* → /equipment/*
│   └── useDialogForm (shared hook)
├── Operations
│   ├── operationsAPI.* → /operations/*
│   └── userAPI.getUsers → /auth/users/public
├── Safety
│   ├── safetyAPI.* → /safety/*
│   └── userAPI.getUsers → /auth/users/public
├── Forecasting
│   ├── forecastingAPI.* → /forecasting/*
│   └── trainingAPI.* → /training/*
├── DocumentExtraction
│   └── documentAPI.* → /document/*
├── Dashboard
│   ├── healthAPI.check → /health/simple
│   ├── equipmentAPI.getAllAssets → /equipment
│   ├── operationsAPI.getTasks → /operations/tasks
│   └── safetyAPI.getIncidents → /safety/incidents
└── Analytics
    ├── equipmentAPI.getAllAssets → /equipment
    └── safetyAPI.getIncidents → /safety/incidents
```

---

# 16. Gaps & Assumptions

1. **No ORM:** The project uses raw SQL via asyncpg with manual query builders. There is no migration framework beyond the SQL files in `data/postgres/migrations/` — migration execution appears manual.

2. **Adapter implementations are stubs:** The WMS, ERP, IoT, and RFID adapters (`src/adapters/`) define interfaces and mock implementations. Real connections to SAP EWM, Manhattan, Oracle, etc. would require vendor-specific configuration.

3. **MCP is custom, not standard:** The MCP implementation is a custom protocol inspired by Anthropic's MCP, not a direct implementation of the open standard. Tool discovery is internal to the application.

4. **Kafka usage is limited:** Kafka is provisioned in Docker Compose but actual producer/consumer code in the backend is minimal — primarily for the document processing pipeline.

5. **GPU features are optional:** GPU-accelerated Milvus (CAGRA index) and RAPIDS forecasting require NVIDIA GPU hardware. The system falls back to CPU implementations.

6. **No WebSocket support:** Chat responses are request-response (not streaming). The frontend simulates streaming with timed UI animations.

7. **Single-tenant:** Auth exists but there is no multi-tenancy or organisation-level isolation. All users share the same data.

8. **Test coverage:** Integration tests exist but many require running infrastructure (DB, Milvus, Redis). Unit tests mock external dependencies.
