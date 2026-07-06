---
title: go-git
---

`go-git` exposes **git repository facts** — the current branch, the origin remote URL, and the repository owner — plus a forward-only check, all read through an injected [`Runner`](https://pkg.go.dev/github.com/gomatic/go-git/facts#Runner) so callers are testable without invoking the real `git` binary. The functionality lives in the [`facts`](https://pkg.go.dev/github.com/gomatic/go-git/facts) package.

- **Source:** [`gomatic/go-git`](https://github.com/gomatic/go-git)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-git/facts](https://pkg.go.dev/github.com/gomatic/go-git/facts)

## Install

```sh
go get github.com/gomatic/go-git
```

## Usage

Everything is driven by a [`Runner`](https://pkg.go.dev/github.com/gomatic/go-git/facts#Runner), which runs a git subcommand and returns its stdout. In production, pass [`NewExecRunner()`](https://pkg.go.dev/github.com/gomatic/go-git/facts#NewExecRunner) — the real-`git` implementation; in tests, pass a fake.

```go
package main

import (
	"fmt"
	"log"

	"github.com/gomatic/go-git/facts"
)

func main() {
	r := facts.NewExecRunner()

	branch, err := facts.Branch(r)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("branch:", branch)

	origin, err := facts.Origin(r)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("origin:", origin)

	owner, err := facts.OwnerOf(r)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("owner:", owner)
}
```

### Functions

- [`Branch(r Runner) (BranchName, error)`](https://pkg.go.dev/github.com/gomatic/go-git/facts#Branch) — the current branch name (`git symbolic-ref --short HEAD`), or `ErrDetachedHead` when HEAD is detached.
- [`Origin(r Runner) (OriginURL, error)`](https://pkg.go.dev/github.com/gomatic/go-git/facts#Origin) — the `remote.origin.url`, or `ErrNoOrigin` when it is not configured.
- [`OwnerOf(r Runner) (Owner, error)`](https://pkg.go.dev/github.com/gomatic/go-git/facts#OwnerOf) — the owning account/org, parsed from the `upstream` remote and falling back to `origin`; handles both `https://` and scp-style (`git@host:owner/repo`) URLs. Returns `ErrNoOrigin` when neither remote yields an owner.
- [`EnsureForwardOnly(r Runner) error`](https://pkg.go.dev/github.com/gomatic/go-git/facts#EnsureForwardOnly) — verifies HEAD is on a branch tip rather than a detached commit (returns `ErrDetachedHead` otherwise).

### Errors

Both sentinels are [`gomatic/go-error`](https://github.com/gomatic/go-error) constants, matched with [`errors.Is`](https://pkg.go.dev/errors#Is) — never by string:

```go
if _, err := facts.Branch(r); errors.Is(err, facts.ErrDetachedHead) {
	// HEAD is detached
}
```

- [`ErrDetachedHead`](https://pkg.go.dev/github.com/gomatic/go-git/facts#pkg-constants) — HEAD is not on a branch tip.
- [`ErrNoOrigin`](https://pkg.go.dev/github.com/gomatic/go-git/facts#pkg-constants) — no origin remote configured.

### Testing with a fake Runner

Because every function takes a `Runner`, tests never touch a real repository — supply canned output:

```go
type fakeRunner struct {
	out facts.CommandOutput
	err error
}

func (f fakeRunner) Run(args ...facts.Arg) (facts.CommandOutput, error) {
	return f.out, f.err
}

func TestBranch(t *testing.T) {
	branch, err := facts.Branch(fakeRunner{out: "main\n"})
	// branch == "main", err == nil
}
```

## Design

- **Injected runner, no hidden globals.** The git binary is reached only through the `Runner` interface; [`ExecRunner`](https://pkg.go.dev/github.com/gomatic/go-git/facts#ExecRunner) is the sole real implementation, so every code path is reachable from a test with a fake.
- **Named domain types.** Arguments and results use dedicated newtypes — `Arg`, `CommandOutput`, `BranchName`, `OriginURL`, `Owner` — rather than bare strings.
- **Value receivers, immutable, private by default.** `ExecRunner` is an empty value type; only what callers need is exported.
- **Extracted from [`skykernel/skym`](https://github.com/skykernel/skym)'s `internal/git`** — generalized and moved to `gomatic` for reuse.
