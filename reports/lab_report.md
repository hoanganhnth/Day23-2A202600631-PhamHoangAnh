# Day 08 Lab Report — LangGraph Agentic Orchestration

## 1. Team / student

- Name: Phạm Hoàng Anh (MSV: 2A202600631)
- Repo/commit: Day23-2A202600631-PhamHoangAnh
- Date: 2026-06-29

## 2. Architecture

The architecture implements a flexible StateGraph workflow with 11 specialized nodes and conditional routing:
- **Intake & Classification**: `intake` normalizes queries, followed by `classify` using Structured Output LLM classification across 5 intents (`simple`, `tool`, `missing_info`, `risky`, `error`).
- **Tool & Retry Loop**: `tool` executes mock operations with transient error simulation. `evaluate` acts as a quality gate. `retry` increments attempt counters bounded by `max_attempts` before escalating to `dead_letter`.
- **HITL & Approval**: `risky_action` prepares high-risk actions, followed by `approval` for human verification before proceeding to execution.
- **Clarification**: `clarify` prompts the user for missing context when queries are vague.
- **Finalization**: All execution branches converge at `finalize` to emit standardized audit events before reaching `END`.

## 3. State schema

| Field | Reducer | Why |
|---|---|---|
| messages | append (`add`) | Audit log of conversation messages across nodes |
| tool_results | append (`add`) | Accumulate tool outputs for grounded generation |
| errors | append (`add`) | Log transient failure messages and exceptions |
| events | append (`add`) | Standardized audit events for grading and metrics |
| route | overwrite | Current classified routing state |
| risk_level | overwrite | Security assessment ('high' vs 'low') |
| attempt | overwrite | Retry attempt counter |
| max_attempts | overwrite | Maximum allowed retry threshold |
| final_answer | overwrite | Grounded response generated for user |
| evaluation_result | overwrite | Quality gate status ('success' vs 'needs_retry') |
| pending_question | overwrite | Clarification question for missing info |
| proposed_action | overwrite | Description of risky action requiring HITL |
| approval | overwrite | Decision details from reviewer |

## 4. Scenario results

### Metrics Summary
- **Total Scenarios**: 7
- **Success Rate**: 100.0%
- **Average Nodes Visited**: 6.4
- **Total Retries**: 3
- **Total Interrupts**: 2

### Per-Scenario Details
| Scenario | Expected route | Actual route | Status | Retries | Interrupts |
|---|---|---|---:|---:|---:|
| S01_simple | simple | simple | PASSED | 0 | 0 |
| S02_tool | tool | tool | PASSED | 0 | 0 |
| S03_missing | missing_info | missing_info | PASSED | 0 | 0 |
| S04_risky | risky | risky | PASSED | 0 | 1 |
| S05_error | error | error | PASSED | 2 | 0 |
| S06_delete | risky | risky | PASSED | 0 | 1 |
| S07_dead_letter | error | error | PASSED | 1 | 0 |

## 5. Failure analysis

1. **Transient Tool Failure & Bounded Retry**: System failures during tool execution are routed to a retry loop. If transient errors persist beyond `max_attempts`, the bounded check in `route_after_retry` escalates execution to `dead_letter`, preventing infinite looping and emitting an audit failure log.
2. **Risky Actions Without Approval**: High-risk operations (e.g., refunds, cancellations) require explicit supervisor verification in `approval_node`. If approval is rejected, the workflow safely diverts to `clarify` rather than executing unauthorized side effects.

## 6. Persistence / recovery evidence

The checkpointer is configured via `persistence.py` supporting both in-memory (`MemorySaver`) and SQLite (`SqliteSaver` with WAL mode). Each scenario run uses a unique `thread_id` (`thread-{scenario_id}`), enabling state persistence, history inspection, and seamless resume capabilities across crashes or interrupts.

## 7. Extension work

- **SQLite Persistence**: Integrated `SqliteSaver` checkpointer adapter with WAL mode for durable state storage across process restarts.
- **Structured Output LLM Classification**: Used Pydantic schema enforcing strict route selection in `classify_node`.

## 8. Improvement plan

1. **Streaming & Parallel Fan-out**: Implement `Send()` API to execute multiple tool calls concurrently for complex queries.
2. **Dynamic Human-in-the-Loop Web UI**: Connect the `interrupt()` mechanism to a Streamlit / Next.js dashboard for real-time human reviews.
