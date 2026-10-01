# Copilot code review instructions

otelc instruments Go programs at build time. `otelc go build` runs a setup phase that matches
YAML rules against the packages in the build, then runs the compiler through `-toolexec`, which
rewrites the AST of matched packages and injects hook code. Contributor rules are in `AGENTS.md`
and `CONTRIBUTING.md`.

## What to look for

Focus on defects a maintainer would block on:

- **Behavior changes for users** of `otelc` or of the instrumented program: altered results,
  swallowed errors, panics, process exits, data races.
- **Silent failures**: a misconfiguration or load failure that is ignored, only logged to
  `.otelc-build/debug.log`, or turned into "nothing was instrumented". Anything the user has to
  act on must be returned as an error or printed to stderr, not only logged.
- **Tests that don't test the change**: the new branch is never reached, state is shared between
  cases or runs, an assertion would still pass with the fix reverted, or a substring check also
  matches wrong output.
- **Documentation drift**: docs that no longer match the code, especially `docs/rules.md` after
  rule schema or matching changes, and doc examples that would not compile.
- **Nondeterminism** in generated code, logs or error messages, such as map iteration order or
  unsorted file lists.

## What not to comment on

CI already enforces these, so don't repeat them:

- Anything `golangci-lint` reports (`.tools/golangci.yml` enables 77 linters, including
  `govet`, `errcheck`, `bodyclose`, `spancheck`, `copyloopvar` and `gosec`), formatting, typos
  and markdown lint.
- License headers, conventional-commit PR titles, freshness of `tool/data/otelc-bundle.tgz`,
  golden files, test file naming and the coverage floor.

Also avoid:

- Comments on the PR title or description.
- Speculative hardening ("a future caller might…") when the PR doesn't introduce the risk. The
  project prefers the simplest change that is correct today.
- Rewriting working code for style alone.

## Check before claiming

- Read the code at the PR head before saying a case is unhandled. Don't flag code that already
  handles it.
- The Go version is the `go` directive in `go.mod`. Loop variables are per-iteration since
  Go 1.22, so don't flag loop-variable capture.
- Check the Go documentation before claiming platform-specific behavior. For example, `os.Rename`
  replaces an existing file on Windows too.
- The semantic conventions version is pinned in `.semconv-version`, and emitted telemetry is
  declared in `schemas/otelc/groups/`. Don't suggest deprecated attribute names.
- Code kept for compatibility with released `otelc` versions is usually deliberate. Look for a
  comment explaining it before suggesting removal.
