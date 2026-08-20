# IV-0001: OpenAI Codex subscription support

## Record

- **Status:** implemented as a single ownership patch
- **Upstream:** [vercel-labs/fx](https://github.com/vercel-labs/fx)
- **Implementation base:**
  [`ac0ce0b64409404b3f35cc264f62c5d14ef1ba37`](https://github.com/vercel-labs/fx/commit/ac0ce0b64409404b3f35cc264f62c5d14ef1ba37)
- **Reference implementation:**
  [earendil-works/pi at `b7bb00b936dbe21b8e160b3e89efdec361846699`](https://github.com/earendil-works/pi/tree/b7bb00b936dbe21b8e160b3e89efdec361846699)
- **Deliverable:** `patches/fx/0001-IV-0001-Add-OpenAI-Codex-subscription-support.patch`

## Purpose

Allow fx users with an eligible ChatGPT subscription to authenticate with
OpenAI Codex and run a Codex model without supplying a separate API key.
Preserve the existing Vercel login and AI Gateway behavior for all other
credentials and models.

The authentication and wire behavior follow pi's implementations of
[OpenAI Codex OAuth](https://github.com/earendil-works/pi/blob/b7bb00b936dbe21b8e160b3e89efdec361846699/packages/ai/src/auth/oauth/openai-codex.ts)
and the
[Codex Responses transport](https://github.com/earendil-works/pi/blob/b7bb00b936dbe21b8e160b3e89efdec361846699/packages/ai/src/api/openai-codex-responses.ts).
The code is adapted to fx's Zig authentication, session, tool, and streaming
interfaces rather than copied as a separate runtime.

## User flow

```bash
fx login openai-codex
fx
fx logout openai-codex
```

`fx login codex` and `fx logout codex` are accepted aliases. Login displays
the OpenAI verification URL and user code, polls until authorization completes,
persists the refreshable session, selects the `openai_codex` credential source,
and sets the model to `openai-codex/gpt-5.6-sol`.

## Design

### Authentication

- OpenAI device authorization is implemented as a distinct provider beside the
  existing Vercel OAuth flow.
- The ID token is decoded to obtain the ChatGPT account ID used by the Codex
  backend.
- Access and refresh tokens are stored in `openai-codex-auth.json`, separate
  from the Vercel credential file.
- The credential file uses fx's verified private-file storage and lock-based
  update path. Token refresh occurs before expiry and rewrites the stored
  session atomically.
- Logout removes only the selected provider's authentication state. Logging out
  of OpenAI Codex also clears the matching credential source and model choice.

### Request routing

The direct transport is selected only when both conditions hold:

1. The active credential source is `openai_codex`.
2. The selected model has the `openai-codex/` prefix.

All other requests retain the upstream Vercel AI Gateway path. The OpenAI
access token is therefore never offered to the gateway model catalog or a
non-Codex model route.

### Responses protocol

- fx messages are converted to Responses API input items.
- System text becomes request instructions.
- Function tools are translated from fx's `inputSchema` shape to Responses
  `parameters`.
- Assistant tool calls and tool outputs preserve their call IDs.
- Server-sent events stream text deltas, reasoning summaries, tool arguments,
  completion state, errors, and usage into fx's existing callbacks.
- ChatGPT account, originator, beta, session, user-agent, and streaming headers
  follow pi's direct Codex transport.
- Prompt cache and session identifiers are bounded to the backend's 64-byte
  header limit.

### Model behavior

The patch defines local capabilities for `openai-codex/gpt-5.6-sol`, including
tools, parallel tool calls, reasoning levels through `xhigh`, and the context
and output limits used by the provider. The direct model remains usable even
though it is intentionally absent from Vercel's public model catalog.

## Files affected

| Area | Files | Responsibility |
| --- | --- | --- |
| Authentication | `src/core/auth/openai_codex.zig`, `auth_runtime.zig`, `credentials.zig`, `oauth_transport.zig` | Device flow, token parsing, secure persistence, refresh, provider dispatch |
| Transport | `src/gateway/openai_codex.zig`, `src/core/agent/stream_provider.zig`, runtime routing files | Request conversion, direct HTTP request, SSE parsing, route isolation |
| CLI | `src/builtins/commands.zig`, `gateway.zig`, `src/core/cli/cli_surface.zig` | Provider selection, help, login and logout behavior |
| Models and UI | `src/core/config/model_capabilities.zig`, shared and presentation files | Local model definition and credential-source presentation |
| Verification | `src/openai_codex_test.zig`, `build.zig` | Focused test root and `test-openai-codex` build step |

## Verification

CI recreates the implementation from the patch on the pinned upstream commit,
then runs:

```bash
zig fmt --check build.zig src
zig build
zig build test-openai-codex --summary all
```

The focused suite covers:

- device polling intervals and JWT account ID extraction
- persisted session round trips
- request schema, tools, and prompt cache fields
- streamed text, reasoning, tool calls, completion, and usage
- local model capabilities
- isolation from the Vercel model catalog

Local verification also runs the built binary's `login --help` path to confirm
the provider-specific CLI surface is linked into the executable.

## Known limitations

- This patch covers the native fx runtime. It does not add a Codex transport to
  fx's WASM or N-API SDK surfaces.
- The direct path supports fx's text, reasoning, and function-tool workflow.
  Image input and structured-output extensions remain on existing provider
  paths.
- Device login and token refresh require network access to OpenAI. Unit tests
  exercise deterministic parsing and conversion boundaries without live
  credentials.

## Maintenance

For an upstream update, check out the new fx revision and apply the mailbox with
`git am`. Resolve conflicts in a temporary upstream checkout, rerun the three CI
commands above, then regenerate the mailbox with `git format-patch` and update
both `BASE` files.

Retire this patch when fx provides equivalent native OpenAI Codex subscription
authentication and direct transport support upstream.
