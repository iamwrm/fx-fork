# IV-0002: OpenCode Zen and OpenCode Go providers

## Record

- **Status:** implemented as a single ownership patch on top of IV-0001
- **Upstream:** [vercel-labs/fx](https://github.com/vercel-labs/fx)
- **Implementation base:**
  [`ac0ce0b64409404b3f35cc264f62c5d14ef1ba37`](https://github.com/vercel-labs/fx/commit/ac0ce0b64409404b3f35cc264f62c5d14ef1ba37)
- **Reference implementation:**
  [badlogic/pi-mono](https://github.com/badlogic/pi-mono) (`packages/ai`
  providers `opencode` and `opencode-go`)
- **Deliverables:** the ordered `patches/fx/*.patch` mailbox series,
  `.github/workflows/ci.yml`, and `.github/workflows/release.yml`

## Purpose

Let fx talk to the two OpenCode model gateways directly with an OpenCode API
key, beside the existing Vercel AI Gateway and OpenAI Codex transports:

| fx model namespace | Gateway | Wire endpoint |
| --- | --- | --- |
| `opencode/<model>` | OpenCode Zen | `https://opencode.ai/zen/v1/chat/completions` |
| `opencode-go/<model>` | OpenCode Go | `https://opencode.ai/zen/go/v1/chat/completions` |
| `<vendor>/<model>` (default) | Vercel AI Gateway | unchanged |
| `openai-codex/<model>` | ChatGPT Codex backend | unchanged |

Both OpenCode fleets accept the same account API key, which is why one
credential source covers both namespaces. `api.opencode.ai` is a different
service and is never used for model traffic.

## User flow

```bash
fx login opencode            # prompts on stdin, or: fx login opencode <key>
fx login opencode-go <key>   # same key storage, different default fleet/model
fx ask "..."                 # uses the saved credential and default model
fx logout opencode           # removes the stored key (shared by both aliases)
```

`zen`, `go`, and `opencode-go` are accepted aliases. The key may also be
supplied through the `OPENCODE_API_KEY` environment variable without any login.
Login stores the key in `~/.fx/opencode-auth.json` (mode 600, verified private
directory, advisory-locked durable replace), selects the `opencode` credential
source, and sets a default model from the chosen fleet.

## Design

### Multi-provider model disambiguation

The same raw model id can exist on several providers (for example
`kimi-k2.7-code` is served by both OpenCode fleets, `gpt-5.6-luna` exists on
OpenCode Go, the Codex subscription, and the AI Gateway). fx resolves this by
making the provider part of the model id:

- Routing happens on the model namespace first, then validates the credential:
  - `opencode/*` or `opencode-go/*` requires the `.opencode` credential;
    anything else fails with `OpenCodeCredentialRequired`.
  - The `.opencode` credential with a non-OpenCode model fails with
    `OpenCodeModelRequired` instead of silently sending gateway models to
    OpenCode.
  - `openai-codex/*` keeps its existing dedicated transport; bare
    `openai/*`, `zai/*`, … ids keep using the AI Gateway.
- Prefixes are distinct byte sequences (`opencode/` vs `openai-codex/` vs
  `openai/`), so no aliasing ambiguity exists.
- In the interactive app, selecting an OpenCode model adopts the OpenCode
  credential when it resolves (environment or stored key), so mixing providers
  works without manually switching credentials. Adoption is best effort; when
  no key resolves the transport error names the missing piece.

### Authentication

- One credential source `.opencode` covers both fleets, mirroring pi where both
  providers read `OPENCODE_API_KEY`.
- Resolution order inside the source: environment variable first, then the
  stored key file. The key is static, so no refresh logic exists.
- The source participates in the standard precedence walk after
  `openai_codex`, appears in the auth hub inventory via `sourceExists`, and is
  labelled `OpenCode API key`.

### Transport

- A single transport module serves both endpoints; the chat URL derives from
  the model namespace.
- Requests are OpenAI Chat Completions JSON: system/user/assistant/tool
  messages, streamed tool calls replayed as assistant `tool_calls`, fx
  `inputSchema` tools converted to `{"type":"function","function":{...}}`,
  `stream_options.include_usage`, and `max_tokens` only when capabilities
  provide an output limit. AI Gateway provider-tool advertisements are dropped,
  as with the Codex transport.
- Streaming parses `chat.completion.chunk` deltas: content, reasoning
  (`reasoning_content`/`reasoning`/`reasoning_text`, first non-empty wins),
  index-keyed tool call accumulation with start/input callbacks, finish reason,
  and usage. SSE comment lines (`: keep-alive`) are skipped. A clean close
  after a terminal `finish_reason` is accepted even without `[DONE]`; an
  unexplained early end raises `OpenCodeStreamEndedEarly` for the retry layer.
- Images and structured responses are rejected as with the initial Codex
  transport.

### Capabilities

OpenCode models advertise tool use and parallel tool calls. Context windows use
a conservative family table (claude 200k, gemini 1M, gpt-5 400k, glm 200k,
kimi 262k, deepseek 128k, grok 256k, minimax 200k, qwen 262k) with unknown
families reporting no window rather than a guess. No reasoning-effort options
are advertised yet because effort control is not uniform across the fleets.

## Verification

- `zig build test-opencode` runs focused authentication, request mapping, SSE
  parsing, and routing tests (namespace collision cases included).
- CI additionally runs the focused suite next to `test-openai-codex`.
- Live checks performed against the real gateways: endpoint reachability
  (401 without a key), credential mismatch errors, a text completion turn, and
  a full agent turn with a streamed tool call executed and answered.

## Known limitations / follow-ups

- Model discovery for the picker still comes from the AI Gateway catalog only;
  OpenCode models are selected by typing `/model opencode/<id>`. A dynamic
  `/v1/models` merge for direct providers is a natural follow-up.
- Credential-mismatch errors render as error names in the CLI surface, matching
  the existing Codex behavior; humanized copy is a shared follow-up.
- Reasoning effort mapping per OpenCode family is not implemented.
