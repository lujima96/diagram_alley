---
phase: 11
slice: D1
spec: specs/D1-ai-generation.md
status: planned
---

# Slice D1 — AI Generation

## Pre-conditions

- [ ] Slice F1 complete — `diagrams`, `diagram_versions`, `model_providers` tables exist.
- [ ] Slice F2 complete — `validate_spec`, `repair_spec`, `render_ascii`, `render_mermaid` are callable.
- [ ] Slice F3 complete — subscription feature gate (`require_active_subscription`) implemented; BYOK encryption available.
- [ ] Slice F5 complete — response envelope, error codes, and rate limiting wired; endpoint stubs registered.

---

## Build Sequence

1. **Provider adapters** — Implement the three provider adapters in `app/providers/`: `openai_compatible.py` (calls `/v1/chat/completions` via `httpx`), `ollama.py` (calls `/api/chat`), `llamacpp.py` (calls `/completion`). Each raises `ProviderError` on failure. All share the `ProviderAdapter` protocol from `base.py`. Decrypt `api_key_encrypted` just-in-time via F3 §4.2 in the factory. (→ D1 §1.2)

2. **Provider CRUD endpoints** — Implement `GET/POST/PATCH/DELETE /api/v1/providers` and `POST /api/v1/providers/{id}/test`. The test endpoint calls the adapter with a minimal prompt and returns `{"data": {"ok": true}}` or the error. Enforce the partial unique index on `is_default` — if a new provider is set as default, clear `is_default` on all other providers for that user first. (→ D1 §1.3, F5 §10)

3. **System prompt templates** — Create `app/services/prompt_templates/` with one `<diagram_type>_system.txt` file per diagram type. Each template instructs the LLM to return valid Diagram Alley JSON only, embeds the F1 §3 schema for the type, and includes one worked example. Implement `prompt_builder.py::build_system_prompt(diagram_type)` that loads and returns the template. (→ D1 §2.1)

4. **Generate endpoint** — Implement `POST /api/v1/diagrams/generate`. Follow the 14-step service flow exactly (D1 §3.2): validate → subscription gate → resolve provider → build prompt → call adapter (1 retry on network failure) → parse JSON → validate spec → repair if needed (max 1 pass) → re-validate → create `diagrams` row → create initial `diagram_versions` snapshot → emit `DIAGRAM_CREATED` and `DIAGRAM_AI_GENERATED` → return. Return `502 PROVIDER_UNAVAILABLE` or `503 PROVIDER_TIMEOUT` on adapter failure; `422 GENERATION_PARSE_FAILED` if response is not parseable JSON; `422 GENERATION_INVALID_SPEC` if spec fails after repair. (→ D1 §3, FM-01, FM-02)

5. **Improve endpoint** — Implement `POST /api/v1/diagrams/{id}/improve`. Follow D1 §4.2 steps: load diagram → subscription gate → build improve prompt with current `spec_json` (truncate to 3000 chars if over token estimate) → call adapter → parse → validate → repair → compute structural diff (D1 §4.3) → return proposed spec + diff **without saving**. Emit `DIAGRAM_AI_IMPROVE_PROPOSED`. Do not persist until the client accepts. (→ D1 §4)

6. **Frontend: prompt panel wiring** — Wire the `<PromptPanel>` stub from slice-F4 to real API calls. Generate button calls `POST /api/v1/diagrams/generate`; on success, redirect to `/workspace/:diagramId` and load the returned spec into `diagramStore`. Improve button calls `POST /api/v1/diagrams/{id}/improve`; on success, show a diff preview panel (new nodes highlighted green, removed nodes red) with Accept and Discard buttons. Accept calls `PATCH /api/v1/diagrams/{id}` with the proposed spec and `create_version_snapshot=true`. (→ D1 §3.3, §4.2)

7. **Provider status indicator** — Wire the sidebar provider status indicator (from F4 §3.2 shell) to `GET /api/v1/providers`. Green dot if at least one provider is configured; grey dot with a link to `/settings/providers` if none. Show "No provider configured" banner in the prompt panel when no default provider exists. (→ D1 §1.3)

8. **Repair/warning banner** — When the generate or improve response includes non-empty `repairs_applied[]` or `warnings[]`, show the yellow repair banner at the top of the Preview panel (described in FM-02). Banner is dismissable. Warnings are not persisted (DEC-031). (→ D1 §3.4, FM-02, DEC-031)

---

## Done Criteria

- [ ] `POST /api/v1/diagrams/generate` with a valid prompt and configured BYOK provider returns a persisted diagram with `ascii_cache` populated. (→ GP-01, GP-02)
- [ ] Generation creates both a `diagrams` row and an initial `diagram_versions` snapshot in the same request. (→ DEC-020)
- [ ] A provider network failure returns `502 PROVIDER_UNAVAILABLE` after one retry; prompt is preserved in UI. (→ FM-01)
- [ ] LLM response not parseable as JSON returns `422 GENERATION_PARSE_FAILED`; no diagram row is created. (→ FM-02)
- [ ] LLM response failing validation after 1 repair pass returns `422 GENERATION_INVALID_SPEC`; no diagram row is created. (→ FM-02)
- [ ] `POST /api/v1/diagrams/{id}/improve` returns a `proposed_spec` and `diff` without modifying `diagrams.spec_json`. (→ D1 §4.2)
- [ ] A user with `status='canceled'` calling generate receives `403 SUBSCRIPTION_REQUIRED`. (→ F3 §5.5)
- [ ] `POST /api/v1/providers/{id}/test` returns `{"ok": true}` for a correctly configured Ollama provider. (→ D1 §1.2)
- [ ] `model_providers` can have at most one `is_default=true` per user — a second default insert raises a constraint error. (→ D1 §1.3, F1 §4.6)
