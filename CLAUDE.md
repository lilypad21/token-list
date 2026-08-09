# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this repository is

Three things live side by side here, and it matters which one a task touches:

1. **A data file** — `src/tokens/solana.tokenlist.json` is the Solana Token
   Registry: ~5,650 SPL token entries plus their logo assets in
   `assets/mainnet/<mint-address>/`. This is what nearly all incoming pull
   requests change.
2. **An npm package** — `@solana/spl-token-registry`, a small TypeScript library
   (`src/lib/tokenlist.ts`) that fetches/filters that list. Published from
   `main` on every push.
3. **A Go bot** — `automerge/`, which validates and bulk-merges token-addition
   PRs on a cron schedule. It is the de facto spec for what a valid token entry
   is.

`origin` is `https://github.com/lilypad21/token-list`, a fork of
`solana-labs/token-list`. Hardcoded URLs throughout the code and data
(resolution strategies, `logoURI` prefixes, the Go module path, CDN purge) all
point at `solana-labs/token-list` — treat those as upstream constants, not
things to rewrite to the fork.

## Layout

```
src/index.ts                      re-exports ./lib/tokenlist
src/lib/tokenlist.ts              the entire library (types + provider + container)
src/lib/tokenlist.spec.ts         ava tests
src/tokens/solana.tokenlist.json  the token registry data (~77k lines)
src/types/.keep                   placeholder; tsconfig typeRoots includes this dir
assets/mainnet/<mint>/<logo>      logo files (png/jpg/svg), one dir per mint address
automerge/automerge.go            the automerge bot: PR fetch, diff parse, validate, commit, push
automerge/parser/parser.go        lenient JSON parser for partial diff hunks ("NormalizeWhatever")
automerge/parser/types.go         Go structs for Token / TokenList
automerge/auth/auth.go            GitHub App installation-token auth
automerge/schema.cue              CUE schema — the authoritative validation rules
validate.sh                       cue vet of the tokenlist against schema.cue
.github/workflows/                build (PR), main (publish), automerge_new (cron), codeql, release
```

## Commands

Package manager is **yarn** (`yarn.lock` is committed; `package-lock.json` is
gitignored). `node_modules` is not checked in — run `yarn` first.

```bash
yarn                 # install
yarn test            # run-s build test:* — build, then lint, prettier check, unit tests
yarn build           # build:main (tsc -> dist/main) + build:module (tsc -> dist/module)
yarn test:unit       # nyc + ava — REQUIRES a prior build (see below)
yarn fix             # prettier --write + eslint --fix over src
```

Schema validation of the data file needs the CUE CLI, which is not a yarn
dependency:

```bash
go install cuelang.org/go/cmd/cue@v0.4.0
PATH=$PATH:~/go/bin ./validate.sh
```

Go bot:

```bash
cd automerge && go build -o /tmp/automerge github.com/solana-labs/token-list/automerge
cd automerge && go test ./...
/tmp/automerge -dryRun -v=1 -max=5      # needs GITHUB_TOKEN or GITHUB_APP_PEM
```

### Test gotchas

- ava is configured with `typescript.rewritePaths: { "src/": "dist/main/" }`, so
  it runs **compiled** output. `yarn test:unit` on its own will fail or use stale
  code unless you built first. Prefer `yarn test`.
- `tokenlist.spec.ts` reads `./src/tokens/solana.tokenlist.json` by relative
  path — run tests from the repo root.
- ava runs with `failFast: true`; the first failure stops the run.
- `test/` is gitignored (a leftover of the `diff-integration-tests` script from
  the project template). There is no integration test directory; don't create
  files there expecting them to be committed.

## Adding or changing tokens

Edit **only** `src/tokens/solana.tokenlist.json` (append to the `tokens` array)
and add the logo under `assets/mainnet/<mint address>/<file>.<png|jpg|svg>`.

Entry shape (2-space indent, matching the surrounding file):

```json
{
  "chainId": 101,
  "address": "<base58 mint address>",
  "symbol": "TKN",
  "name": "Token Name",
  "decimals": 9,
  "logoURI": "https://raw.githubusercontent.com/solana-labs/token-list/main/assets/mainnet/<mint>/logo.png",
  "tags": ["utility-token"],
  "extensions": { "website": "https://example.com" }
}
```

`chainId` is the cluster: `101` mainnet-beta, `102` testnet, `103` devnet
(mirrors the `ENV` enum in `src/lib/tokenlist.ts`). `tags` must be keys already
declared in the top-level `tags` map (`stablecoin`, `ethereum`, `lp-token`,
`wrapped-sollet`, `wrapped`, `leveraged`, `bull`, `bear`, `nft`,
`security-token`, `utility-token`, `tokenized-stock`) — note that the data
itself already uses tags outside that map (e.g. `wormhole`, `social-token`), and
the CUE schema only constrains tag *format*, not membership.

### What the automerge bot enforces

From `automerge/automerge.go` and `automerge/schema.cue`. A PR is auto-merged
only if all of this holds:

- **Additions only.** Any removed or modified line in the diff rejects the PR.
  The sole exception is deletion of a line whose non-space content is exactly
  `{`, `}`, `[`, or `]`.
- **Only these files may change:** `src/tokens/solana.tokenlist.json` and new
  files under `assets/mainnet/<mint>/`. Changes to `CHANGELOG.md` and
  `package.json` are ignored; anything else rejects the PR.
- **Assets must be new** (no modifications), exactly 4 path segments
  (`assets/mainnet/<mint>/<file>`), extension `.png`/`.jpg`/`.svg` (any case),
  and **under 200 KiB**. Every asset must correspond to a token added in the
  same PR, and vice versa the `logoURI` must resolve.
- **Uniqueness** of `address` and of `name` per `chainId`, both against the
  existing list and within the PR itself.
- **`logoURI` is required** for new tokens (`#StrictTokenInfo`) and is verified:
  if it points at `raw.githubusercontent.com/solana-labs/token-list/main/assets/…`
  it must match a file added in the PR; otherwise it must return HTTP 200 to a
  HEAD request. **The filename in the JSON must match the actual asset
  filename** — a mismatch is the most common failure.
- **`extensions.website`** must return 200 to a HEAD request;
  **`extensions.coingeckoId`** must resolve at
  `https://www.coingecko.com/en/coins/<id>`.
- **Schema conformance** per `schema.cue`: base58 address (43–44 chars),
  `decimals` 0–255, `name` ≤50 chars against a restricted charset, `symbol`
  ≤20 chars alphanumeric plus `+-%/$_`, ≤10 tags, `#URL` scheme must be
  `http`/`https`/`ipfs`, extension keys are a closed set. Non-conforming
  historical entries are grandfathered via the `#SymbolWhitelist` and
  `#NameWhitelist` unions — **append to those lists only for existing data, never
  to let a new token through.**
- Name prefix `SOLKITTY NFT` and websites under `https://solkitty.io/nft` are
  blacklisted in code.

Outcome is reported as a check run named `New automerge` plus an `automerge` or
`automerge-error` label. The bot commits each accepted PR separately (authored
as the PR submitter, message `<PR title>\n\nCloses #<n>`), force-pushes the
result to the `automerge-pending` branch, and a separate workflow step
fast-forwards `main` via `git merge --no-ff`. The bot refuses to force-push to
`main`/`master`.

When the bot writes the file it re-serializes with `json.MarshalIndent(…, "", "  ")`
through the Go `Token` struct, so field order and formatting are normalized on
merge. Match the existing style in hand edits rather than inventing your own.

### Note on current data state

The most recent entry in the list (`FT4nC3F1Nm2KwNyANcggnuiLSgeBkV7eNoZTmRZrimxJ`,
symbol `URCK`) has a malformed `logoURI` — the scheme reads `htps://`. That
fails the `#URL` pattern in `schema.cue`, so `./validate.sh` will not pass
until it is corrected. Fix it if a task touches that entry; don't silently
"fix" unrelated entries in a PR that is meant to add a token, since unrelated
edits are exactly what the automerge bot rejects.

## Library code conventions

`src/lib/tokenlist.ts` is the whole library. Its shape:

- `TokenListProvider.resolve(strategy)` → `TokenListContainer`. Strategies:
  `CDN` (default, jsDelivr), `GitHub` (raw), `Solana` (token-list.solana.com),
  `Static` (the bundled JSON, imported directly via `resolveJsonModule`).
  Network strategies fall back to the bundled JSON on any error.
- `TokenListContainer` is immutable: `filterByTag`, `filterByChainId`,
  `excludeByTag`, `excludeByChainId`, `filterByClusterSlug`, `getList()`. Each
  filter returns a **new** container — a test asserts this.
- Interfaces use `readonly` members throughout. Class methods are written as
  arrow-function properties. Keep both conventions.
- `TokenExtensions` in TypeScript is a *narrower* set of keys than
  `#Extensions` in `schema.cue`, and the Go parser uses
  `map[string]string`. If you add an extension key to the data, decide
  deliberately whether the TS interface and the CUE schema both need it — the
  CUE schema is a closed struct and *will* reject unknown keys.

### Style

- Prettier with `singleQuote: true` (config lives in `package.json`);
  `package.json` itself is prettier-ignored.
- ESLint: `@typescript-eslint/recommended` + prettier, with enforced
  `import/order` (alphabetized, newlines between groups) and `sort-imports` for
  member sorting. Import ordering violations fail CI.
- TypeScript is `strict`, plus `noUnusedLocals`, `noUnusedParameters`,
  `noImplicitReturns`, `noFallthroughCasesInSwitch`.
- `.editorconfig`: LF, UTF-8, 2-space indent, final newline, 80 cols (tabs /
  4-width for Go files).
- cspell runs over the repo with the word list in `.cspell.json`; add new
  project-specific vocabulary there rather than disabling the check.
- Commit messages follow Conventional Commits (commitizen +
  `cz-conventional-changelog`), though the merged history is dominated by
  bot-authored `Adding <Token>` commits.

## CI and release

| Workflow | Trigger | Does |
| --- | --- | --- |
| `build.yml` | pull_request | `yarn && yarn test`, then `cue vet` via `validate.sh` |
| `main.yml` | push to `main` | test, `standard-version --patch`, push tags, purge jsDelivr CDN, `npm publish`. Guarded by `github.repository_owner == 'solana-labs'`, so it does not run on this fork |
| `automerge_new.yml` | hourly cron (`10 * * * *`) + manual | build and run the Go bot, then merge `automerge-pending` into `main` |
| `codeql_analysis.yml` | push/PR to `main`, weekly | CodeQL for JavaScript |
| `release.yml` | tag `v*` | create a GitHub release |

Versions in `package.json` and entries in `CHANGELOG.md` are generated by
`standard-version` in CI. **Do not hand-edit the version field or the
changelog** — and note that the automerge bot explicitly ignores diffs to those
two files.

## Working here

- A token addition is a data change. Don't refactor library code, reformat the
  JSON, or touch unrelated entries in the same change — the automerge bot
  rejects unrelated diffs, and that rule is the point of the repo's workflow.
- Before claiming a data change is valid, run `./validate.sh` (installing `cue`
  if needed) and `yarn test` — the test suite checks the file parses as JSON
  and contains no duplicate mainnet addresses.
- The library has no runtime dependencies beyond `cross-fetch`. Keep it that
  way; it ships to browsers.
- Develop on the branch you were assigned, commit with a descriptive message,
  and push with `git push -u origin <branch>`. Don't open a pull request unless
  asked.
