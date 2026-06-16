# Architecture: Fold list-form system content into Responses API `instructions`

This document records the *how* for `system-instructions-responses-api.spec.md`.
The spec is the source of truth for what/why; this file holds the resolved
design and the signature manifest. It does not modify the spec.

## Signature

- spec_path: specs/system-instructions-responses-api/system-instructions-responses-api.spec.md
- algorithm: sha256 (shasum -a 256), architecture.md excluded
- files:
  - system-instructions-responses-api.spec.md: a40d246ae64d703b023d53cb2140eab7e42d0e1dbfaef76354d087026ada9137
- combined: 524f5d72e4d3f6ee68275182b5a4c335a8ad9d453caa0766bf60a642782a7f82

Re-verify with:

```
find specs -type f ! -name architecture.md | sort | xargs shasum -a 256
find specs -type f ! -name architecture.md | sort | xargs shasum -a 256 | shasum -a 256
```

The per-file line must match the `files:` entry; the second command's output
must match `combined`.

## 1. Chosen approach (Variant A, confirmed)

Fix lives entirely in
`litellm/completion_extras/litellm_responses_transformation/transformation.py`,
in `LiteLLMCompletionResponsesConfig.convert_chat_completion_messages_to_responses_api`
(method body at transformation.py:222-308; the system branch is lines 234-252).

The `role == "system"` branch is rewritten so that **both** string-form and
list-form system content are folded into the accumulated `instructions` string,
and **no** `role: system` item is ever appended to `input_items`. List-form
content is reduced to text in-place (iterate blocks, take `type == "text"`
text, warn-and-skip everything else) rather than delegating to
`_convert_content_to_responses_format`, because that helper produces
Responses-API content blocks (`input_text`/`input_image`/...) suited for an
`input` message, not a flat instruction string, and it never emits warnings for
dropped blocks. Reusing it here would reintroduce the bug or require
post-processing its output; a small local extractor is simpler and matches FR-3.

### Rejected alternatives

- **Fix in `chatgpt/` provider (`transform_responses_api_request`).** Rejected:
  the spec explicitly scopes to Variant A. It would duplicate logic per
  Responses-API provider and leave the same 400 for any other Responses backend
  (volcengine, xai, openai responses). The root cause is provider-agnostic and
  belongs in the shared chat to Responses conversion.
- **Fix in the Anthropic adapter (`_add_system_message_to_messages`) so it emits
  a string.** Rejected: that adapter already correctly produces an OpenAI
  `role: system` message with list content (legitimate OpenAI shape, used to
  preserve `cache_control`); flattening there would lose information for
  non-Responses backends and only patches the Anthropic to Responses path, not
  the general OpenAI-chat to Responses path. Out of scope per spec.
- **Reuse `_convert_content_to_responses_format` then stringify its output.**
  Rejected: double transformation, no per-block warning, and couples the
  instruction extractor to the message-content format. More code, more coupling.

## 2. Contract / signature (unchanged)

```python
def convert_chat_completion_messages_to_responses_api(
    self, messages: List["AllMessageValues"]
) -> Tuple[List[Any], Optional[str]]:
```

The public signature and return contract `(input_items, instructions)` are
**unchanged**. Both callers consume the tuple positionally and are unaffected:

- `transformation.py:403` (`transform_request` path).
- `litellm/proxy/guardrails/guardrail_hooks/noma/noma.py:188`, which only reads
  `instructions` and folds it into a single `input_text` system message. The
  separator change (space -> `\n\n`) is cosmetically irrelevant to noma's logic
  but does change one of its assertions (see section 5).

Post-conditions strengthened by this change (invariants downstream may rely on):
- The returned `input_items` contains no element with `role == "system"` for
  any system message received (FR-6), regardless of string vs list content.
- `instructions` is `None` iff no system text was extracted from any system
  message (FR-7); otherwise it is the `\n\n`-joined accumulation of all system
  text in original message order.

## 3. System-branch design (sketch, not final code)

Replace the entire `if role == "system":` block (transformation.py:234-252).
The string path's separator changes from a single space to `\n\n` (NFR-5).

```python
if role == "system":
    extracted = self._extract_system_instruction_text(content)
    if extracted:
        instructions = (
            f"{instructions}\n\n{extracted}" if instructions else extracted
        )
    continue  # never append a role: system item (FR-6, FR-7)
```

New private helper on the same class:

```python
def _extract_system_instruction_text(
    self, content: Any
) -> str:
    if isinstance(content, str):
        return content
    if not isinstance(content, list):
        return ""  # error-table row: neither str nor list -> no-op, no crash
    texts: List[str] = []
    for block in content:
        if isinstance(block, dict) and block.get("type") == "text":
            texts.append(block.get("text", ""))  # missing text -> "" (table row)
        else:
            block_type = (
                block.get("type") if isinstance(block, dict) else type(block).__name__
            )
            verbose_logger.warning(
                "litellm responses transformation: dropping non-text system "
                f"content block of type {block_type!r} from instructions"
            )
    return "\n\n".join(t for t in texts if t)
```

Notes that map to the spec error table and FRs:
- FR-1/FR-2: only `type == "text"` blocks contribute; joined with `\n\n` in
  order.
- FR-3 / NFR-4: every non-text block (and every non-dict block) emits exactly
  one `verbose_logger.warning` naming the dropped type; extraction continues.
- FR-7 / AC-6: empty list or all-non-text list yields `""`, so the caller adds
  nothing to `instructions` and appends no input item.
- Error table "block missing `text` key": `block.get("text", "")` yields `""`,
  which is filtered out of the join (the trailing
  `if t` guard) so it never produces a stray empty paragraph.
- Error table "block is not a dict": falls to the `else`, warns with the
  Python type name, skipped.
- `content` neither str nor list: returns `""` (no-op), preserving "do not
  crash"; this is a behavior-neutral choice because the previous code path for
  that case (the old `else` -> `_convert_content_to_responses_format`) was the
  buggy one being removed, and the spec's error table specifies no-op.

`continue` is used instead of falling through so it is structurally impossible
for a system message to reach the `elif content is not None` message-appending
branch. This is the mutation-resistant guarantee for FR-6.

The `f"{instructions}\n\n{extracted}"` accumulation deliberately mirrors the
list and string paths through the **same** join convention so block-form and
string-form system messages interleave correctly (AC-5).

## 4. Open questions from spec section 8 — resolved

- **Logging channel (`verbose_logger.warning`).** Confirmed correct.
  `verbose_logger` is already imported at transformation.py:28
  (`from litellm._logging import verbose_logger`) and used throughout this module
  (e.g. the `debug` calls inside `_convert_content_to_responses_format`).
  `warning` level surfaces under `--detailed_debug` and in normal runs at
  WARNING without being chatty in the hot path (one line only per dropped
  block). No reason to use `litellm.utils` print-style logging. Decision: use
  `verbose_logger.warning`.

- **Existing test asserting the single-space multi-system join.** Found exactly
  one, and it is the one the spec's risk note anticipated:
  `tests/test_litellm/proxy/guardrails/guardrail_hooks/test_noma.py`,
  `test_pre_call_hook_with_multiple_system_prompts` (lines 548-604). Its
  assertion at lines 596-599 expects
  `"You are a helpful assistant You should be polite and respectful"` (single
  space). After the separator change this must become
  `"You are a helpful assistant\n\nYou should be polite and respectful"`. This
  is a required, deliberate update (NFR-5) and must be called out in the PR
  description, not changed silently.

  No other test asserts the space-join through the target function. Adjacent
  hits checked and cleared:
  - `tests/.../test_responses_adapters_transformation.py:728-746` exercises a
    *different* code path (the Anthropic responses adapter
    `translate_request`, which joins list system blocks with a single `\n`).
    Not affected by this change; do not touch.
  - `tests/.../test_openai_count_tokens_transformation.py:119-147` exercises
    `OpenAICountTokensConfig.messages_to_responses_input`, a different function;
    its system cases are single-message. Not affected.
  - The mapped test file has only one `role: system` usage
    (test file line 1860), a single string system message inside a
    `transform_response` test; it asserts nothing about the join. Not affected.

## 5. Tests

Mapped test file (per CLAUDE.md `tests/test_litellm/` mirror rule):
`tests/test_litellm/completion_extras/litellm_responses_transformation/test_completion_extras_litellm_responses_transformation_transformation.py`.

Add unit tests that call
`LiteLLMResponsesTransformationHandler().convert_chat_completion_messages_to_responses_api(...)`
directly and assert on the returned `(input_items, instructions)`:

- AC-1: one list system message with a single text block -> `instructions`
  equals that text; assert no item in `input_items` has `role == "system"`.
- AC-2: list system message `[{text:"A"},{text:"B"}]` ->
  `instructions == "A\n\nB"`; no system item.
- AC-3: list system message `[{type:"image",...},{type:"text",text:"X"}]` ->
  `instructions == "X"`; assert the `verbose_logger.warning` fired and names the
  dropped type. Capture the logger via `caplog`
  (`caplog.at_level(logging.WARNING, logger="LiteLLM")`) or by injecting a
  spy/`patch("....transformation.verbose_logger")` and asserting on
  `warning.call_args`. **Do not** monkeypatch a class attribute (CLAUDE.md).
- AC-4: string system message -> folded into `instructions` unchanged; no system
  item. (Locks FR-5.)
- AC-5: two system messages, string `"A"` then list `[{text:"B"}]` ->
  `instructions == "A\n\nB"`; no system item. (Locks the cross-form join and the
  separator simultaneously.)
- AC-6: list system message with no text blocks (empty list, and separately a
  list of only non-text blocks) -> `instructions is None` and no system item.
- AC-7 (regression guard): a conversation with user, assistant,
  assistant-with-tool-calls, and tool messages produces `input_items`
  byte-for-byte identical to a snapshot captured from current behavior. The
  simplest mutation-resistant form: build the message list, capture the
  expected `input_items` literal once, assert equality. Confirms only the system
  branch changed.

Mutation-kill requirements baked into the assertions:
- Reverting the fix (system list -> `role: system` input item reappears) must
  fail AC-1/AC-2/AC-5/AC-6 via the "no `role == 'system'` in input_items" + the
  `instructions` value assertions.
- Regressing the separator from `\n\n` back to a space must fail AC-2 and AC-5
  (they assert the literal `"\n\n"`), and the updated noma test.
- Dropping the warning must fail AC-3.

Existing test to update (required, not optional):
- `tests/test_litellm/proxy/guardrails/guardrail_hooks/test_noma.py`,
  `test_pre_call_hook_with_multiple_system_prompts`, assertion at lines 596-599:
  change the expected text from the single-space join to the `\n\n` join
  `"You are a helpful assistant\n\nYou should be polite and respectful"`. Flag
  in the PR body as the deliberate NFR-5 behavior change.

## 6. Reuse pointers for coder

- Logger: `verbose_logger` already imported (transformation.py:28). No new
  import.
- Do **not** route system content through `_convert_content_to_responses_format`
  (transformation.py:824); that helper is for `input` message content, emits no
  drop warnings, and returns block dicts, not a string.
- Keep the helper on `LiteLLMCompletionResponsesConfig` next to
  `convert_chat_completion_messages_to_responses_api`; prefer
  `insert_before_symbol`/`replace_symbol_body`-style edits.
- No new config flags, no comments unless an invariant truly needs one
  (CLAUDE.md). The `continue` + single-join convention should be self-evident.

## 7. Residual risks / assumptions left for implementation

- The `developer` role is **not** in scope. Today it falls through to the
  message branch and becomes an `input` message item; this change does not touch
  it. If a future Responses backend also rejects `developer` in `input`, that is
  a separate spec. Coder should not opportunistically extend the system branch
  to `developer`.
- `cache_control` on Anthropic system text blocks is dropped when folding into
  `instructions` (the Responses `instructions` field has no cache_control slot).
  This is inherent to the target format and already true for the string path;
  the spec accepts it implicitly (AC-1 routes a `cache_control`-bearing Claude
  Code system prompt and only asserts the text reaches `instructions`). No
  action; noted so it is not mistaken for data loss.
- Proof-of-fix (live Codex 200) is an end-to-end gate owned by the
  implementation/verification stages, not reproducible at design time; AC-1's
  unit-level half (no `role: system` item, text in `instructions`) is fully
  covered by the tests above, and the spec already documents the curl runbook in
  its TODO checklist.
