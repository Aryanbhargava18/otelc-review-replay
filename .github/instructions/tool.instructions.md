---
applyTo: "tool/**"
---

# Reviewing `tool/`

`otelc` has to see the build exactly the way the `go` command it wraps does. Most defects found in
review here are places where the two disagree.

## Fidelity with the go command

- New `packages.Load` calls must get the user's build flags (`-tags`, `-mod`, `-modfile`,
  `-overlay`, `-C`). Loading without them gives wrong results for tag-gated or vendored code.
- `-C` is normalized into `packages.Config.Dir` by `loadDirFromBuildFlags` in
  `tool/internal/pkgload`. New code should reuse it instead of parsing `-C` again. `go` changes to
  the `-C` directory first, so relative paths in other flags resolve against it.
- On the command line, value flags come as `-flag value` or `-flag=value`. In `GOFLAGS` only
  `-flag=value` is valid, and entries are split with `cmd/internal/quoted.Split` semantics (see
  `tool/util/go.go`): any whitespace, including `\n` and `\r`, separates entries, and quotes have
  no escapes.
- `go test` accepts its own flags (`go help testflags`) and `--`. Argument-splitting changes have
  to handle both.
- `packages.Load` returning a nil error doesn't mean every package loaded. Check
  `len(pkg.Errors) > 0` for each package that is used.
- The compiler may remap imports through the importcfg `importmap` (for example vendored
  `vendor/golang.org/x/net/...` paths in the standard library). Lookups by import path have to
  apply the map.
- Each `packages.Load` or `go list` call starts a subprocess. Flag new per-item calls inside loops
  that could be batched, and repeated loads of the same directory.

## Rules (`tool/internal/rule`, `tool/internal/setup`)

- Invalid rule YAML must fail at load time with an error that names the rule. It must never be
  accepted and then silently match nothing. Check null list entries, empty strings, case or
  whitespace variants of special values such as `$root`, YAML aliases, and fields that have to
  agree, such as `path` and `module`.
- Every place that parses rule YAML, such as the setup loader and the matcher in
  `tool/internal/setup/pin.go`, must accept and reject the same input.
- A rule must not be applied twice when several roots or globs match the same package.
- Changes to the rule schema or matching need a matching update to `docs/rules.md` in the same PR.

## AST rewriting (`tool/internal/instrument`, `tool/internal/ast`)

- Generated identifiers and import qualifiers must be valid at the insertion point. Check
  shadowing by parameters and locals, aliased imports (`import f "fmt"`), and one path imported
  under two aliases. Reporting a conflict is better than generating code that compiles but binds
  to the wrong thing.
- A rewrite must only touch code the rule introduced, not user code embedded into the same
  template through `{{ . }}`.
- Cloned AST nodes have no entry in the parser's position map. Code that looks up positions after
  a replacement has to account for that.

## Errors and exits

- Create errors with `ex.New` or `ex.Newf`, wrap standard-library and third-party errors once
  with `ex.Wrap` or `ex.Wrapf`, and then just `return err`. Flag re-wrapping that repeats context
  the message already has.
- `ex.Fatal`, `ex.Fatalf`, `util.Assert` and `util.AssertType` exit the process. They must not be
  reachable from user input such as rule files, flags or the source being compiled; return an
  error instead.

## Tests

- Golden tests in `tool/internal/instrument/testdata/golden/` should cover the new behavior, and
  each fixture's comments and `rules.yml` should describe what it actually tests.
- Tests of stateful setup steps should use fresh state for each simulated `otelc` run, the way a
  second invocation would.
