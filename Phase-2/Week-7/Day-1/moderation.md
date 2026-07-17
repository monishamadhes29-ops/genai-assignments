# Moderation Layer Integration Plan (AI-Shield)

Status: **Phases 1-7 implemented and verified live.** Phase 9 (image/screenshot content moderation via OCR) added and verified.

## Overview

This document specifies how the Test Case Generator RAG backend/frontend integrate with **AI-Shield**, an external rule-based moderation API gateway (`AI-Shield.postman_collection.json`, `baseUrl = http://localhost:4000`). AI-Shield runs as a separate service from this backend (`backend/.env` → `PORT=5005`), so integration means this app becomes an **HTTP client** of AI-Shield — no detector logic is implemented in this repo.

Source requirements: `IntegrationPlanprompt.md`.

Mandatory rules driving this plan:
1. API changes are implemented and verified **before** any UI work.
2. Every request that would reach the LLM must first pass through moderation.
3. If AI-Shield returns a **block** verdict, the LLM must **not** be called.
4. The moderation integration must be toggleable on/off via `.env`.
5. A blocked request returns a **user-friendly error** with guidance, including the **reason and detector details**.

Phase numbers below are independent of AI-Shield's own internal phase labels (its Postman collection's "Phase 5–13" folders describe how AI-Shield itself was built, not this integration).

---

## Endpoint Contracts (verified live against a running AI-Shield instance)

Only two endpoints are called — the combined pipeline for text, and the file validator for uploads. The individual per-detector endpoints (`/api/detect/pii`, `/cii`, `/secret`, `/toxic`, `/injection`, `/token`) exist in AI-Shield but are **not** called directly; `/api/moderate` already runs that full pipeline internally.

### `POST /api/moderate` — combined text pipeline

| | |
|---|---|
| Request | `{ "text": string, "model"?: string }` |
| Response status | `200` for `ALLOW`/`MASK`, **`422`** for `BLOCK`, `400` if `text`/`file` missing |
| Top-level `action` | `ALLOW`, `MASK`, `BLOCK` |

Actual response shape:
```json
{
  "status": "success",
  "action": "MASK",
  "detectorResults": [
    { "detector": "pii", "triggered": true, "action": "MASK",
      "findings": [{ "type": "email", "value": "john.doe@gmail.com", "masked": "[EMAIL_REDACTED]", "position": {"start":12,"end":30} }] },
    { "detector": "secret", "triggered": false, "action": "ALLOW", "findings": [] },
    { "detector": "token", "triggered": false, "action": "ALLOW", "findings": [], "message": "Token count within limit: 16/4096" }
  ],
  "sanitizedContent": "My email is [EMAIL_REDACTED] and my Aadhaar is [AADHAAR_REDACTED]",
  "originalContent": "My email is john.doe@gmail.com and my Aadhaar is 2345 6789 0123",
  "metadata": { "tokenEstimate": 16, "processingTimeMs": 0, "timestamp": "..." }
}
```
Key finding: **`MASK` verdicts return `200` with a ready-to-use `sanitizedContent` field** — no manual redaction/position-offset reconstruction is needed. `findings[].value` carries the *raw, unredacted* matched text (e.g. the actual email address) — this must never be forwarded to clients or logged in full; only `type` and `masked` are safe to surface.

### `POST /api/detect/file` — file validator

| | |
|---|---|
| Request | `multipart/form-data`, field `file` |
| Response status | `200` for `ALLOW`, `422` for `BLOCK` |
| Example | `.exe` → `BLOCK` (`invalid_extension`/`invalid_mime_type`); `.csv` → `ALLOW` |

---

## Phase 1 — Configuration & Toggle ✅

**`backend/.env`** / **`backend/src/config/env.ts`**:
- `MODERATION_ENABLED` (default `false`) → `env.moderationEnabled: boolean`, via a new `optionalBool()` helper alongside the existing `required()`/`optional()`.
- `MODERATION_BASE_URL` (default `http://localhost:4000`) → `env.moderationBaseUrl`.
- `MODERATION_TIMEOUT_MS` (default `5000`) → `env.moderationTimeoutMs`.
- `MODERATION_FAIL_MODE` (`open` | `closed`, default `closed`) → `env.moderationFailMode`.
- `validateEnv()` fails fast at startup if `MODERATION_ENABLED=true` with no `MODERATION_BASE_URL`, or an invalid `MODERATION_FAIL_MODE`.

---

## Phase 2 — Moderation Client ✅

**`backend/src/services/moderationClient.ts`**
- `moderateText(text, model?)` → `POST /api/moderate`. Returns `{ verdict, sanitizedText?, detectors }`; short-circuits to `ALLOW` when `MODERATION_ENABLED=false`.
- `moderateFile(buffer, filename)` → `POST /api/detect/file` (native `FormData`/`Blob`, no new dependency needed — Node 18+/`@types/node` provide global `fetch`/`FormData`/`Blob`).
- Both use an `AbortController` timeout (`moderationTimeoutMs`).
- `ModelNameFor()` (exported from `modelFactory.ts`) supplies the active provider's real model name to `/api/moderate`'s `model` field for accurate token-limit checks.

---

## Phase 3 — Error Contract ✅

Two distinct error types, because "a detector rejected this content" and "AI-Shield itself is down" are different problems for a client to react to:

- **`ModerationBlockedError`** — a detector actually triggered `BLOCK`. → **`422`**:
  ```json
  {
    "blocked": true,
    "error": "Your request was blocked by our content moderation system.",
    "reason": "Request blocked: prompt injection detected",
    "guidance": "Remove or rephrase the flagged content ... and try again.",
    "detectors": [
      { "name": "injection", "verdict": "BLOCK", "message": "Request blocked: prompt injection detected",
        "findings": [{ "type": "ignore_instructions" }, { "type": "role_switch" }] }
    ]
  }
  ```
  Findings are redacted to `{ type, masked }` only — raw `value`/`position` are stripped before this ever leaves the backend.
- **`ModerationUnavailableError`** — AI-Shield unreachable/timed out and `MODERATION_FAIL_MODE=closed`. → **`503`**, `{ "error": "The content moderation service is currently unavailable." }`.
- `sendModerationErrorResponse(err, res)` in `moderationClient.ts` is the single shared handler reused by all three routes.

---

## Phase 4 — Gate Text Requests ✅

- `testCaseChain.ts` → `chatGenerateTestCases` (the single choke point behind `/api/chat` and both `/api/generate-testcases` paths) calls `moderateText(input.message, modelNameFor(...))` before the generation retry loop and before touching session history.
- On `MASK`, generation proceeds using AI-Shield's `sanitizedContent` in place of the raw message — the redacted text is what's sent to the LLM *and* what's persisted into session history, so PII/secrets never reach the model or the conversation memory.
- On `BLOCK`, `ModerationBlockedError` propagates straight out — `invokeChain` (and therefore the LLM) is never called.
- `chatRoute.ts` / `testCaseRoute.ts` catch blocks call `sendModerationErrorResponse` before falling back to the generic `502`.

**Verified live:** injection payload → `422` with detector detail, no LLM call; clean text → real LLM response returned.

---

## Phase 5 — Gate File Uploads ✅

- `uploadRoute.ts` calls `moderateFile(file.buffer, file.originalname)` after the existing type/size checks and before any parsing (`parsePdf`/`parseDocx`/`parseExcel`/`toImagePayload`).

**Verified live:** a valid PDF passes moderation and parses normally with the gate active. (Disallowed extensions like `.exe` are already rejected earlier by this app's own `uploadConfig.ts` allowlist before moderation is even reached — AI-Shield's file check adds defense-in-depth against MIME/extension spoofing within the allowed kinds.)

---

## Phase 6 — Toggle & Failure-Mode Behavior ✅

| `MODERATION_ENABLED` | AI-Shield reachable? | `MODERATION_FAIL_MODE` | Result |
|---|---|---|---|
| `false` | n/a | n/a | No moderation calls made. |
| `true` | yes | n/a | Normal gating per Phases 4–5. |
| `true` | no / timeout | `closed` | `ModerationUnavailableError` → `503`. LLM not called. |
| `true` | no / timeout | `open` | Warning logged, request proceeds as allowed. |

**Verified live** (isolated test against an unreachable port): `closed` → blocked with `ModerationUnavailableError`; `open` → allowed through with a logged warning.

---

## Phase 7 — UI Integration ✅

**Goal:** Surface the `422` block reason/detector detail and the `503` unavailable state to the user.

- `frontend/src/services/api.ts` — `ModerationBlockedError extends ApiError` (`reason`/`guidance`/`detectors`), detected via a shared `errorFromBody()` helper used by both the `fetch`-based calls and the XHR upload path.
- `frontend/src/components/ChatWindow.tsx` — a distinct red "Blocked by content moderation" bubble (reason + guidance + per-detector findings) instead of the generic error bubble.
- `frontend/src/components/FileUpload.tsx` — surfaces the same block reason + detector names in the upload chip.
- Styling (`frontend/src/App.css`) reuses existing danger/error tokens; no new `.env`/toggle needed on the frontend — moderation is enforced entirely server-side.

**Verified live in a browser:** injection payload → red blocked bubble with reason + detector detail, no LLM call; clean text → real generation succeeds; a non-moderation error (LLM provider rate limit) renders as the distinct pre-existing generic error bubble, proving the two paths never collide.

---

## Phase 9 — Image/Screenshot Content Moderation (OCR) ✅

**Problem:** AI-Shield has no endpoint for analyzing actual image pixel content — only text detectors and a file-type/extension validator (`/api/detect/file`, already gating uploads since Phase 5). A screenshot showing a real secret/API key or PII visible in the UI would reach the vision model completely unchecked.

**Approach:** OCR any visible text out of the screenshot and run it through the existing `/api/moderate` text pipeline — reuses every detector already integrated, no changes to AI-Shield itself, and matches this app's actual risk (a UI screenshot leaking visible sensitive data), not generic image-safety classification (out of scope / not what this tool's screenshots are).

- **New dependency:** `tesseract.js` (pure JS/WASM OCR, no native binary/system dependency).
- **New file:** `backend/src/documents/parsers/imageTextExtractor.ts` — `extractTextFromImage(dataUrl)` decodes the data URL and OCRs it via `tesseract.js`'s `createWorker("eng")`.
- **`backend/src/services/moderationClient.ts`** — `moderateText()` gained an optional `sourceLabel` param that prefixes a BLOCK reason (e.g. `"Text detected in the uploaded screenshot: ..."`), so the existing blocked-bubble UI distinguishes an image-content block from a typed-message block with zero frontend changes.
- **`backend/src/chains/testCaseChain.ts`** — in `chatGenerateTestCases`, alongside the existing message-text moderation (before the retry loop, still the single choke point before any LLM call): if `env.moderationEnabled` and an image is attached, OCR it and — only if text was actually found — moderate that text separately via `moderateText(ocrText, modelNameFor("vision"), "Text detected in the uploaded screenshot")`.
- **Design limitations, by choice:**
  - **BLOCK-only signal.** AI-Shield's `sanitizedContent` (for a `MASK` verdict) can't be reapplied to image pixels — there's no redaction/inpainting. A `MASK` or `ALLOW` verdict on the OCR'd text changes nothing; only `BLOCK` stops the request, same as the file-type validator's existing ALLOW/BLOCK-only contract.
  - **OCR failures fail open**, independent of `MODERATION_FAIL_MODE` (which governs AI-Shield *reachability*, not local OCR reliability): a corrupt image or OCR error is logged and treated as "no text found," not a block — OCR is a best-effort enhancement layered on top of the file-type check and message-text moderation, not the sole safety net.
  - **Skipped entirely when `MODERATION_ENABLED=false`** — no OCR latency cost (~1-1.5s per image) paid when moderation is off.

**Verified:**
- OCR extraction confirmed against a real generated PNG containing an email address — text extracted correctly.
- Full `/api/chat` round-trip with an image attached succeeded end-to-end (AI-Shield was down at test time, confirming the fail-open path also works correctly for this new call).
- `BLOCK` verdict + `sourceLabel` prefixing confirmed against a local mock AI-Shield returning a `secret` detector block — reason correctly read `"Text detected in the uploaded screenshot: Request blocked: secrets detected in payload"`.

---

## Phase 8 — Verification

- ✅ Replayed representative Postman collection payloads directly against AI-Shield and through this app's `/api/chat`, `/api/generate-testcases`, `/api/upload`.
- ✅ Confirmed `MODERATION_FAIL_MODE` open/closed behavior with AI-Shield unreachable.
- ✅ Browser verification of Phase 7 UI (block, allow, error-vs-block distinction).
- ✅ Phase 9 (image OCR moderation) verified per above.
