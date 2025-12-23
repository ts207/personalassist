# personalassist
State model (v1) — entities, tables, transitions, invariants
Assumptions (explicit)

Single user initially, but schema supports multiple users.

“Session” = one conversation thread.

Each user message triggers a Run; a Run may include multiple Model Calls and Tool Calls (loop).

Rolling summary is optional; schema supports both “full log” and “summary + last N”.

Tools are executed by backend; model only requests tool calls.

1) Entities and relationships (high level)

User 1—N Session

Session 1—N Message

Message(user) 1—N Run (typically 1 run per user message)

Run 1—N ModelCall

ModelCall 0—N ToolCall

User 1—N PreferenceProfile (usually 1 active row)

User 1—N MemoryItem

User 1—N Document

Document 1—N DocChunk (embedded chunks for RAG)

Run 0—N CitationRef (what sources supported final answer)

ToolCall 0—1 ConfirmationRequest (only for write/action tools)

2) Relational schema (SQL-ish, implementation-agnostic)
2.1 Users

users

id (UUID/ULID, PK)

created_at, updated_at

display_name (nullable)

status enum: active|disabled

timezone default "America/Toronto" (optional)

Indexes:

PK on id

2.2 Sessions (conversation threads)

sessions

id (UUID, PK)

user_id (FK users.id, indexed)

created_at, updated_at

title (nullable)

status enum: active|archived

settings_json (JSON) — per-session overrides (strict verified mode, allow_web, memory_on, etc.)

rolling_summary (TEXT, nullable)

summary_updated_at (nullable)

Indexes:

(user_id, updated_at DESC)

Notes:

Store “strict verified mode” here (session-level), with defaults from user preferences.

2.3 Messages (canonical chat log)

messages

id (UUID, PK)

session_id (FK sessions.id, indexed)

user_id (FK users.id, indexed)

created_at

role enum: user|assistant|system|tool

content (TEXT) — final text shown in UI (for tool messages store summarized content)

content_json (JSON, nullable) — structured payloads (tool results, structured outputs)

reply_to_message_id (FK messages.id, nullable) — threading

run_id (FK runs.id, nullable) — which run produced this assistant message

visibility enum: user_visible|internal default user_visible

Indexes:

(session_id, created_at)

(run_id)

Notes:

Keep “tool” role messages if you want a complete trace inside the same stream; otherwise rely on tool_calls table + tool_trace UI.

2.4 Runs (one orchestration cycle per user input)

runs

id (UUID, PK)

session_id (FK sessions.id, indexed)

user_id (FK users.id, indexed)

created_at, updated_at

trigger_message_id (FK messages.id) — the user message that started this run

status enum:

queued

running

awaiting_confirmation

completed

failed

mode enum: normal|strict_verified

allow_web_search (bool)

memory_enabled (bool)

max_tool_iters (int)

error_code (nullable)

error_detail (nullable text)

final_assistant_message_id (FK messages.id, nullable)

Indexes:

(session_id, created_at)

(status)

Why Runs matter:

They are your unit of traceability, cost measurement, and debugging.

2.5 Model calls (each call to Responses API)

model_calls

id (UUID, PK)

run_id (FK runs.id, indexed)

created_at

provider default openai

model (string)

request_json (JSON) — sanitized (redact secrets)

response_json (JSON) — raw (or redacted raw)

output_text (TEXT, nullable) — extracted assistant text

stop_reason (nullable)

tokens_in (int, nullable)

tokens_out (int, nullable)

latency_ms (int, nullable)

call_stage enum:

initial (pre-tools)

tool_followup (after tool results)

final (final answer pass)

memory_gate (post-processing pass)

Indexes:

(run_id, created_at)

2.6 Tool calls (requested by model, executed by backend)

tool_calls

id (UUID, PK)

run_id (FK runs.id, indexed)

model_call_id (FK model_calls.id, indexed)

created_at, updated_at

tool_name (string)

side_effect enum: none|writes_state|external_action

requires_confirmation (bool)

status enum:

requested (model asked)

blocked_policy (backend refused)

awaiting_confirmation

executing

succeeded

failed

args_json (JSON) — validated input

result_json (JSON, nullable)

result_summary (TEXT, nullable) — short text injected back to model

error_code (nullable)

error_detail (nullable)

duration_ms (nullable)

Indexes:

(run_id, created_at)

(tool_name, status)

Important:

This table is the authoritative audit record for tools.

2.7 Confirmation requests (for write/action tools)

confirmation_requests

id (UUID, PK)

tool_call_id (FK tool_calls.id, unique)

run_id (FK runs.id, indexed)

created_at, resolved_at (nullable)

status enum: pending|approved|rejected|expired

confirmation_token (string, unique, indexed)

prompt_text (TEXT) — what the user sees (“Approve creating task X?”)

expires_at (nullable)

Backend behavior:

If tool requires confirmation and token not present → create confirmation_request, set tool_call status awaiting_confirmation, set run status awaiting_confirmation.

2.8 Preferences (explicit profile)

preference_profiles

id (UUID, PK)

user_id (FK users.id, indexed)

created_at, updated_at

is_active (bool)

strict_verified_default (bool)

require_citations_for_facts (bool)

allow_web_search_default (bool)

memory_enabled_default (bool)

verbosity enum: low|medium|high

format_prefs_json (JSON) — e.g., “step-by-step”, “no filler”, etc.

2.9 Memory items (durable, selective)

memory_items

id (UUID, PK)

user_id (FK users.id, indexed)

created_at, updated_at

category enum: preference|profile_fact|project_fact

statement (TEXT) — canonical single sentence

confidence (float 0–1)

status enum: active|inactive|retracted

source_run_id (FK runs.id, nullable)

last_confirmed_at (nullable)

expires_at (nullable)

tags_json (JSON, nullable)

Rule:

Only store items that pass “memory gate” criteria.

2.10 Documents + RAG chunks

documents

id (UUID, PK)

user_id (FK users.id, indexed)

created_at, updated_at

title (string)

source_type enum: upload|note|chat_export|web_snapshot

source_uri (nullable string) — path/object key

doc_date (nullable date) — when written (not upload time)

status enum: active|deleted

metadata_json (JSON)

doc_chunks

id (UUID, PK)

document_id (FK documents.id, indexed)

user_id (FK users.id, indexed)

chunk_index (int)

text (TEXT)

embedding_vector (vector type or external reference)

token_count (int, nullable)

created_at

metadata_json (JSON) — page number, headings, etc.

Indexes:

(user_id, document_id, chunk_index)

vector index on embedding_vector (if supported)

2.11 Citations / provenance (support for “verified facts only”)

citation_refs

id (UUID, PK)

run_id (FK runs.id, indexed)

created_at

source_type enum: web|document|tool_output

source_id (string) — url, document_id, tool_call_id, etc.

locator (nullable) — page, line range, snippet id

quote (nullable TEXT, keep short)

used_in enum: final_answer|intermediate|memory_gate

This lets you enforce: “final answer claims must have citations”.

3) State transitions (runtime)
3.1 Run lifecycle

queued → running

running:

if model returns final answer → completed

if tool call requires confirmation → awaiting_confirmation

if tool call executes and loop continues → remain running

awaiting_confirmation:

user approves (token) → back to running

user rejects → either completed (with refusal/alternative) or failed depending on design

token expires → failed or completed with “expired”

Any state → failed on unrecoverable error

3.2 Tool call lifecycle

requested (from model)

backend decision:

blocked_policy OR

awaiting_confirmation OR

executing

then succeeded or failed

4) Invariants (must always hold)

No write/action tool executes without approval

If requires_confirmation=true then status cannot become executing unless there exists confirmation_requests.status=approved.

Each Run has exactly one trigger user message

runs.trigger_message_id must reference a messages.role=user in same session.

Final assistant message is linked

If runs.status=completed, then final_assistant_message_id is non-null and references an assistant message.

Provenance is captured when strict mode is on

If runs.mode=strict_verified, then either:

citation_refs exist for final answer, or

assistant output explicitly indicates insufficient evidence (enforceable via evals)

Memory items are auditable

Any memory created automatically must reference source_run_id and include confidence.

5) Minimal v1 subset (to start coding safely)

Implement first:

users

sessions

messages

runs

model_calls

Then add:

tool_calls + confirmation_requests

Then add:

documents + doc_chunks (RAG)

Then add:

memory_items + preference_profiles + citation_refs

This sequencing lets you ship a working chat loop before adding complexity.
