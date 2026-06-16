# Spec: Fold list-form system content into Responses API `instructions`

## 1. Overview and user value

When a client speaks the Anthropic Messages API to the LiteLLM proxy (`POST /v1/messages`) and the route targets a Responses-API backend (e.g. the `chatgpt/` provider backed by the ChatGPT subscription Codex endpoint), a system prompt sent as an **array of content blocks** is currently rejected upstream with `400 BadRequestError: ChatgptException - {"detail":"System messages are not allowed"}`.

Claude Code always sends `system` as an array of text blocks (with `cache_control`), not a string. That array is converted into an OpenAI-format `role: system` message whose `content` is a **list**. In the chat-to-Responses transformation, only string-form system content is folded into the `instructions` field; list-form system content instead becomes a `{"type": "message", "role": "system", ...}` item inside `input`. The Codex backend forbids any `system` role in `input`, so the request fails.

User value: Claude Code (and any Anthropic-Messages client that emits block-form system prompts) can drive Responses-API models through the proxy without 400s. The system prompt reaches the model as `instructions`, which is the field Responses-API backends expect.

### Root cause (verified)

1. `LiteLLMAnthropicMessagesAdapter._add_system_message_to_messages` (`litellm/llms/anthropic/experimental_pass_through/adapters/transformation.py:981-997`): array-form `system` becomes `ChatCompletionSystemMessage(role="system", content=<list>)`.
2. `LiteLLMCompletionResponsesConfig.convert_chat_completion_messages_to_responses_api` (`litellm/completion_extras/litellm_responses_transformation/transformation.py:234-252`): for `role == "system"`, only `isinstance(content, str)` is folded into `instructions`; the `else` branch appends a `role: system` item into `input_items`.
3. `ChatGPTResponsesAPIConfig.transform_responses_api_request` (`litellm/llms/chatgpt/responses/transformation.py`): merges only the `instructions` field; `input` passes through untouched via `allowed_keys`, so the stray `role: system` item reaches Codex.

## 2. Scope

In scope:
- Fix is confined to `convert_chat_completion_messages_to_responses_api` in `litellm/completion_extras/litellm_responses_transformation/transformation.py` (Variant A). This closes the gap for every Responses-API provider at once, with no duplicated logic in `chatgpt/`.

Out of scope:
- No changes to the `chatgpt/` provider, the Anthropic adapter, or the proxy config.
- No change to the existing string-form system path's behavior other than what is required for consistent merging (see FR-4).
- No new config flags.

## 3. Functional requirements (EARS format)

- **FR-1** When `convert_chat_completion_messages_to_responses_api` processes a message with `role == "system"` whose `content` is a list of content blocks, the system shall extract the text from each `type == "text"` block and fold the combined text into `instructions` rather than appending a `role: system` item to `input`.

- **FR-2** Where a system message's list content contains multiple text blocks, the system shall join those blocks' text with a double newline (`\n\n`) in their original order.

- **FR-3** Where a system block in the list has a type other than `text`, the system shall skip that block and emit a warning via `verbose_logger` identifying the dropped block type; the request shall still proceed with the text that was extracted.

- **FR-4** Where multiple system messages (or a previously accumulated `instructions` value) are present, the system shall append newly extracted system text to the existing `instructions`, joined with a double newline (`\n\n`).

- **FR-5** When a system message's `content` is a string, the system shall continue to fold it into `instructions` (existing behavior preserved).

- **FR-6** When the transformation finishes, the system shall produce no `input` item with `role == "system"` for any system message it received, regardless of whether that message's content was a string or a list.

- **FR-7** Where extracting text from a list-form system message yields no text (e.g. the list is empty or contains only non-text blocks), the system shall not append anything to `instructions` and shall not append a `role: system` item to `input`.

## 4. Non-functional requirements

- **NFR-1 (Correctness/compat):** Non-system roles (`user`, `assistant`, `tool`, assistant-with-tool-calls) must transform exactly as before; only the system branch changes.
- **NFR-2 (Maintainability):** Reuse the existing `instructions` accumulation variable and join convention; do not introduce a parallel code path. No new comments unless a non-obvious invariant requires one (per repo CLAUDE.md).
- **NFR-3 (Performance):** Single linear pass over system blocks; no added per-message allocation beyond the joined string. No measurable latency impact.
- **NFR-4 (Observability):** Dropped non-text blocks are surfaced through `verbose_logger.warning` so silent data loss is detectable in `--detailed_debug` runs.
- **NFR-5 (Consistency note):** The string-form path currently joins multiple system messages with a single space (`f"{instructions} {content}"`). FR-4 standardizes the merge separator to `\n\n`; the string path will be updated to the same separator so block-form and string-form accumulation behave identically. This is the only behavioral change to the string path.

## 5. Acceptance criteria (Given/When/Then)

- **AC-1 (the bug fix):**
  Given an Anthropic-Messages request whose `system` is an array of one text block routed to a `chatgpt/` (Responses-API) model,
  When the proxy transforms it to a Responses-API request,
  Then the request contains the block's text in `instructions` and contains no `role: system` item in `input`, and the live Codex call returns 200 instead of 400.

- **AC-2 (multiple text blocks):**
  Given a system array with two text blocks `["A", "B"]`,
  When transformed,
  Then `instructions` contains `"A\n\nB"` and `input` has no `role: system` item.

- **AC-3 (non-text block dropped with warning):**
  Given a system array containing one `type: text` block and one `type: image` (or any non-text) block,
  When transformed,
  Then only the text is folded into `instructions`, a `verbose_logger` warning naming the dropped type is emitted, and the request proceeds.

- **AC-4 (string path preserved):**
  Given a system message whose content is a plain string,
  When transformed,
  Then it is folded into `instructions` exactly as before and produces no `role: system` input item.

- **AC-5 (mixed string + list system messages):**
  Given two system messages, one string `"A"` and one list `[{type:text,text:"B"}]`,
  When transformed,
  Then `instructions` equals `"A\n\nB"` (joined with `\n\n`) and no `role: system` item appears in `input`.

- **AC-6 (empty/no-text system list is a no-op):**
  Given a system message whose list content has no text blocks,
  When transformed,
  Then `instructions` is unchanged and no `role: system` item is added to `input`.

- **AC-7 (no regression for other roles):**
  Given a conversation with user, assistant, assistant-with-tool-calls, and tool messages,
  When transformed,
  Then the resulting `input` items are byte-for-byte identical to the pre-change output.

## 6. Error handling

| Scenario | Handling |
|----------|----------|
| System list contains a non-`text` block (e.g. `image`, future type) | Skip the block; `verbose_logger.warning(...)` with the dropped `type`; continue (FR-3). |
| System list is empty or has only non-text blocks | No-op: nothing added to `instructions`, no `input` item (FR-7). |
| System block is a dict missing `text` key | Treat as empty text for that block (extract `""`); contributes nothing to the join. |
| System block is not a dict | Skip and warn, same as non-text block. |
| `content` is neither str nor list (unexpected) | Preserve current fallthrough behavior; do not crash. |

## 7. Implementation TODO checklist

- [ ] In `convert_chat_completion_messages_to_responses_api`, replace the `else` branch under `role == "system"` (transformation.py:242-252) with logic that: iterates list blocks, collects `text` from `type == "text"` blocks, warns on other block types via `verbose_logger`, joins collected text with `\n\n`, and folds the result into `instructions`.
- [ ] Update the string-form merge (transformation.py:237-239) to join with `\n\n` so block-form and string-form accumulation are consistent (NFR-5).
- [ ] Ensure no `role: system` item can be appended to `input_items` anywhere in the system branch (FR-6).
- [ ] Add regression tests in `tests/test_litellm/` covering AC-1 through AC-7 (unit-level on `convert_chat_completion_messages_to_responses_api`; assert `instructions` value and absence of `role: system` in returned `input_items`).
- [ ] Add a test asserting the `verbose_logger` warning fires for a dropped non-text block (AC-3) via dependency-injected/captured logger, not monkeypatching a class attribute (per CLAUDE.md).
- [ ] Run mutation check mentally/locally: tests must fail if the fix is reverted (system item reappears) and if the separator regresses to a space.
- [ ] Format + lint (`make` targets / project tooling) before commit.
- [ ] Proof of fix: run the proxy with `config.stage05.chatgpt.apikey.yaml`, hit `/v1/messages` against `claude-gpt` with a real Claude Code-style block system prompt, show the 200 response (real Codex call). Capture the curl + output for the PR per CLAUDE.md.

## 8. Open questions / risks

- The string-path separator change (NFR-5) is a deliberate, minor behavior change. If any existing test asserts the single-space join for multiple string system messages, it must be updated to `\n\n`; flag it in the PR description rather than silently changing it.
- Confirm `verbose_logger` is the right channel (it is already imported in this module) versus `litellm.utils` print-style logging; warning level chosen so it shows under `--detailed_debug` without spamming default runs.
