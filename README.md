# fx-fork

Patch set adding OpenAI Codex subscription and OpenCode (Zen / Go) provider
support to [vercel-labs/fx](https://github.com/vercel-labs/fx).

This is intentionally a patches-and-documentation repository, not a GitHub
fork and not a copy of the upstream source tree. The patch mailbox in
`patches/fx/` is the source of truth. CI checks out the pinned upstream commit,
applies the patch with `git am`, compiles fx, and runs the focused unit tests.

See [IV-0001](docs/IV-0001-openai-codex-subscription.md) for the Codex design
and [IV-0002](docs/IV-0002-opencode-providers.md) for the OpenCode providers,
including how the same model name on several providers is disambiguated.

## Apply the patch

```bash
git clone https://github.com/iamwrm/fx-fork.git
git clone https://github.com/vercel-labs/fx.git fx
git -C fx checkout ac0ce0b64409404b3f35cc264f62c5d14ef1ba37
git -C fx am "$(pwd)"/fx-fork/patches/fx/*.patch
cd fx
zig build test-openai-codex --summary all
zig build test-opencode --summary all
```

## Use it

```bash
./zig-out/bin/fx login openai-codex
./zig-out/bin/fx
```

`fx login codex` is accepted as a short alias. Sign-out is explicit:

```bash
./zig-out/bin/fx logout openai-codex
```

The login flow uses OpenAI's device authorization flow and selects
`openai-codex/gpt-5.6-sol`. Subscription traffic goes directly to the ChatGPT
Codex Responses endpoint, never through Vercel AI Gateway.

### OpenCode Zen and OpenCode Go

```bash
./zig-out/bin/fx login opencode            # key read from stdin, or pass it:
./zig-out/bin/fx login opencode-go <key>   # same account key, Go fleet default
```

Both fleets accept the same OpenCode API key; `OPENCODE_API_KEY` also works
without any login. Models are namespaced by provider — `opencode/kimi-k3`,
`opencode-go/mimo-v2.5`, `openai-codex/gpt-5.6-sol`, or a bare AI Gateway id —
so the same model name served by several providers is never ambiguous.

## Releases

Release `v0.0.4-codex.3` provides `fx-linux-x86_64.tar.gz` and
`fx-macos-aarch64.tar.gz`, plus a SHA-256 file for each archive. Both packages
include the fx binary, `LICENSE`, and `THIRD_PARTY_NOTICES.md`.

The Codex request hotfixes keep the optional Vision tool from blocking
text-only turns, accept supported `openai/` model aliases such as
`openai/gpt-5.6-luna`, and omit AI Gateway-only provider tool advertisements
that the direct ChatGPT Codex endpoint cannot execute.

The macOS arm64 executable is ad-hoc signed but is not Apple-notarized.
Background auto-upgrade and `fx upgrade` are disabled in patched builds so an
official upstream binary cannot silently replace the Codex transport.

## Patch layout

- `patches/fx/BASE`: the upstream ref used to materialize a checkout
- `patches/fx/BASE_COMMIT`: the exact upstream commit CI requires
- `patches/fx/*.patch`: ordered `git format-patch` mailboxes
- `docs/IV-*.md`: durable design and maintenance records
- `.github/workflows/release.yml`: native release builds and GitHub publication
- `checkouts/`: optional local working trees, ignored by Git
