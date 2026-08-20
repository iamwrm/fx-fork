# fx-fork

Patch set adding OpenAI Codex subscription authentication and transport support
to [vercel-labs/fx](https://github.com/vercel-labs/fx).

This is intentionally a patches-and-documentation repository, not a GitHub
fork and not a copy of the upstream source tree. The patch mailbox in
`patches/fx/` is the source of truth. CI checks out the pinned upstream commit,
applies the patch with `git am`, compiles fx, and runs the focused unit tests.

See [IV-0001](docs/IV-0001-openai-codex-subscription.md) for the design,
security boundaries, implementation map, and verification details.

## Apply the patch

```bash
git clone https://github.com/iamwrm/fx-fork.git
git clone https://github.com/vercel-labs/fx.git fx
git -C fx checkout ac0ce0b64409404b3f35cc264f62c5d14ef1ba37
git -C fx am "$(pwd)"/fx-fork/patches/fx/*.patch
cd fx
zig build
zig build test-openai-codex --summary all
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

## Patch layout

- `patches/fx/BASE`: the upstream ref used to materialize a checkout
- `patches/fx/BASE_COMMIT`: the exact upstream commit CI requires
- `patches/fx/*.patch`: ordered `git format-patch` mailboxes
- `docs/IV-*.md`: durable design and maintenance records
- `checkouts/`: optional local working trees, ignored by Git
