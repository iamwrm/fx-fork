# fx-fork

Patch set adding OpenCode (Zen / Go) provider support to
[vercel-labs/fx](https://github.com/vercel-labs/fx), rebased onto upstream
v0.0.5. Codex subscription support is no longer patched in: upstream ships it
natively since v0.0.5 (see
[IV-0001](docs/IV-0001-openai-codex-subscription.md) for the retirement
record).

This is intentionally a patches-and-documentation repository, not a GitHub
fork and not a copy of the upstream source tree. The patch mailbox in
`patches/fx/` is the source of truth. CI checks out the pinned upstream commit,
applies the patch with `git am`, compiles fx, and runs the focused unit tests.

See [IV-0001](docs/IV-0001-openai-codex-subscription.md) for the Codex design
and retirement record, and [IV-0002](docs/IV-0002-opencode-providers.md) for
the OpenCode providers and their v0.0.5 adaptation.

## Apply the patch

```bash
git clone https://github.com/iamwrm/fx-fork.git
git clone https://github.com/vercel-labs/fx.git fx
git -C fx checkout df7e6245e1992758d4060c97477ceafa27770551
git -C fx am "$(pwd)"/fx-fork/patches/fx/*.patch
cd fx
zig build test-opencode --summary all
```

## Use it

```bash
./zig-out/bin/fx login codex      # native upstream Codex subscription
./zig-out/bin/fx login opencode   # key read from stdin, or pass it:
./zig-out/bin/fx login opencode-go <key>   # same account key, Go fleet default
```

Sign-out is explicit:

```bash
./zig-out/bin/fx logout opencode
```

OpenCode models are namespaced by fleet — `opencode/kimi-k3`,
`opencode-go/mimo-v2.5` — while Codex/Grok use raw ids from their
authenticated catalogs. Upstream's provider switching keeps credentials,
models, and sessions independent per provider, so the same model name served by
several providers is never ambiguous.

Both fleets accept the same OpenCode API key; `OPENCODE_API_KEY` also works
without any login.

## Releases

Release `v0.0.6-codex.1` (first release based on upstream v0.0.5) provides
`fx-linux-x86_64.tar.gz` and `fx-macos-aarch64.tar.gz`, plus a SHA-256 file for
each archive. Both packages include the fx binary, `LICENSE`, and
`THIRD_PARTY_NOTICES.md`.

Upstream's native Codex transport replaces the previous fork implementation.
The macOS arm64 executable is ad-hoc signed but is not Apple-notarized.
Background auto-upgrade and `fx upgrade` remain disabled in patched builds so
an official upstream binary cannot silently remove the OpenCode providers.

## Patch layout

- `patches/fx/BASE`: the upstream ref used to materialize a checkout
- `patches/fx/BASE_COMMIT`: the exact upstream commit CI requires
- `patches/fx/*.patch`: ordered `git format-patch` mailboxes
- `docs/IV-*.md`: durable design and maintenance records
- `.github/workflows/release.yml`: native release builds and GitHub publication
- `checkouts/`: optional local working trees, ignored by Git
