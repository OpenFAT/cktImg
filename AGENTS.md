# AGENTS.md

cktImg: a Schemify plugin that imports a SPICE netlist as editable schematics.
Two halves: the layout engine in `engine/` (Zig 0.17, C ABI, also a Zig
package and the `cktimg-tex`/`cktimg-json` tools) and the plugin at the root
(Rust 1.95, edition 2024, built to `cktimg.wasm`). `README.md` and
`engine/README.md` are the overview.

## Skills to load first

Load each with the Skill tool (other agents: read
`~/.claude/skills/<name>/SKILL.md` and the companion files it names). Source:
https://github.com/OmarSiwy/NixOSConfig/tree/main/skills

| Skill | Use it for here |
|---|---|
| `data-oriented-design` | Netlist, placement and wire tables (`engine/src/netlist.zig`, `placed.zig`, `csr.zig`): the six questions, SoA, narrowest widths, handles |
| `codebase-clean-code` | Owners: the engine lays out, the plugin maps cells and writes Schemify records; one spine per import; seams on the cheapest rung |
| `api-design` | Every `pub` function, `engine/src/abi.zig` and `engine/include/cktimg.h` (shipped surface): usage code first, caller owns memory and flow |
| `code-style` | File order, ≤500 code lines per file, `check_style.py` |
| `simd-loops` | Hot loops in placement and routing cost: the plain-loop forms, or `@Vector` with a scalar oracle |
| `doc-comments` | Every `pub` item, Zig and Rust |
| `ponytail` (ultra) | How little code; never what shape the data takes |
| `assertions` | Contracts, capacities, bounded loops; the plugin's fuel budget per event |
| `test-plan`, `tdd` | The oracles below, the CI gates, a feature's definition of done |

Language:

| Skill | Use it for here |
|---|---|
| `zig` | `engine/`: Zig 0.17 idioms and the zero-cost probe |
| `rust` | The plugin: types, memory, `zero-cost.md`; it runs as a wasm guest, so size counts (550 KB cap) |

Project-specific:

| Skill | Use it for here |
|---|---|
| `compiler-construction` | `engine/src/netlist.zig` and `src/spice.rs`: the SPICE reader, rows plus `extra_data`, interning, coded diagnostics |
| `gui-architecture` | What the plugin hands Schemify's editor: records the view draws, never editor state |
| `branchless` | Data-dependent branches in layout scans, before `simd-loops` |

## The method

Binding on every change, in every project.

### Code

- Five commitments: lean storage, no runtime residue, cheap change,
  contained failure, proof by test.
- Answer the six data questions before writing code. Then: one array per
  field, the narrowest width the stated range allows, typed `u32` handles
  with a sentinel, existence tables instead of flags, hot/cold split, tables
  sized once from a named capacity, a compile-time size assertion on every
  hot struct.
- An abstraction on a hot path carries an assembly probe: FOLDED, IDENTICAL,
  EQUIVALENT or NEGATIVE pass; RESIDUE fails the build. A "no residue" claim
  cites the probe, a speed claim cites a benchmark. Dispatch per batch, never
  per element.
- A seam names what varies and needs a second case today; it sits on the
  cheapest rung (concrete, free function, table, closed enum with exhaustive
  switch, comptime, request table, function-pointer table, `dyn`, plugin).
- Tables, transforms, one spine per pipeline (ifs up, fors down), one owner
  writes each table and everyone else appends requests. Three likely changes
  each have a change radius of zero; the compiler lists every other site.
- APIs start from the caller's code. The caller owns memory and flow:
  requests instead of callbacks, an immediate twin for anything retained.
  Factor at the second real instance.
- Branches: delete, then branchless, then vectorize along a lane axis. The
  `W == 1` instantiation is the scalar oracle; vectorization is verified in
  the assembly.
- GPU: SoA always, the roofline ceiling named with its arithmetic before
  tuning, data kept on the device, fixed-order reductions, the dependency DAG
  recovered before porting. "This belongs on the CPU" is a valid verdict.
- An operating error returns to the caller that can decide; a bug hits an
  assertion. Two or more split, paired assertions per function, a bound on
  every loop, input validated once at the edge, fallible work before the
  commit.
- Pass order: boundary, placement, data, zero-cost proof, contracts, tests,
  branches, lanes. Data layout beats brevity. Every pass keeps NaN gates,
  allocation-failure paths, `errdefer`s, trust-boundary validation and
  tolerances.

### Testing

- The oracle is named before the code and never recomputes the answer the
  way the code does. Cheapest first: assertions, a reference
  implementation, a round trip, a certificate, a metamorphic relation, a
  model, a golden with recorded provenance, a spec example.
- The main bug class is a wrong number with no crash, which coverage and
  fuzzing cannot see. A bug found by a person is a hole in the oracle set:
  it lands with its regression test and the oracle or generator that would
  have caught its class.
- A new test goes red on the bug before it is trusted green.
- Unsupported fails and never skips. A known failure is a named marker in
  the tree; an unexpected pass fails until the marker goes. Name lists are
  compared, never counts.
- A tolerance follows the oracle's own accuracy. Determinism is a test:
  1 job and N jobs give the same bytes, and the order is fixed, not the test.
- A flaky test is fixed or quarantined the same day, never retried
  silently.
- Tiers: pre-commit under 10 s, merge under 15 min, nightly, release. A
  release ships on numbers a script checks. The map lives in `TESTING.md`.

### Builds

- A build is a cache keyed on its declared inputs. Every producing step has
  an output argument, every checking step an exit-code check, every file
  read at run time is declared (`addFileInput`) or the step says
  `has_side_effects`. Keys hash contents, never absolute paths.
- Cut the artifact graph where edits land: one object per generated unit,
  narrow imports for shared ones, kernels out of host graphs.
- Zig edit loop: `zig build check --watch -fincremental`, one watch per
  artifact across all worktrees; the result that counts comes from a plain
  build. One global cache per toolchain store path; a worktree symlinks its
  `.zig-cache` to the main checkout's.
- Rust: flags live in `.cargo/config.toml`, never a one-off `RUSTFLAGS`;
  `check`, `build`, `test` and rust-analyzer agree on features, profile and
  target directory (rust-analyzer gets its own).
- Nothing collects a cache. Keep a byte budget per cache and delete old
  entries only while no build runs (the `long-term-garbage-collection`
  skill); a warm cache is kept, never wiped by routine.
- Gate on the exit code: `cmd | tail` swallows it, so use
  `exit "${PIPESTATUS[0]}"`.

### Maintaining

- `main` is green by machine: work lands through a pull request whose merge
  result passed the required checks. No `dev` branch; topic branches are
  deleted after the merge.
- Bots never push to `main`. Generated output ships as a release asset or
  through a gated pull request.
- Every dependency is pinned by commit and content hash, the toolchain by
  `flake.lock`, actions by SHA. A third-party input you hash is mirrored
  somewhere you control.
- The changelog is written by hand, one line per change citing its issue,
  every number in it measured. A release refuses a version mismatch, builds
  every target, checks every artifact and smokes the bundle.
- CI holds: the merge gate, every target compiled, docs that execute,
  reference suites gated by name, a nightly, the downstream contracts, the
  release job.

## Building

- Every build runs inside `nix develop -c …`: it pins rustc 1.95, wasm-ld and
  Zig 0.17, the toolchain `cktimg.wasm` is byte-reproducible under.
- `./build.sh` rebuilds `cktimg.wasm`, which is committed. Commit it with any
  change to `src/` or `engine/`: CI fails when the committed module differs
  from its rebuild, or reaches 550 KB.
- The plugin: `cargo test --locked`, `cargo fmt --check`,
  `cargo clippy --locked --all-targets -- -D warnings`.
- The engine, from `engine/`: `zig build test` with `-Dlatex_renderer=false`
  and `=true` (the option gates code, both must pass), `zig fmt --check`, and
  `zig build gallery -- --svgs out/` to eyeball a layout change.
- Run one cargo or zig process per agent, and don't build while `free -g`
  shows under 6 GB available.

## Testing

- Every test names its oracle: the independent thing that says the output is
  wrong. The main one is the round trip: an imported fixture, netlisted back
  through Schemify's `elab`, netlister and `spicecmp`, is the same circuit.
- A golden that disagrees with an independent oracle is a bug in the code
  that wrote it. Fix the code, regenerate the golden, and say so in the commit.

## Committing

Commit at every green step: the tests above passing, both formatters clean,
`cktimg.wasm` rebuilt. The message says what changed and in which half. Never
commit red to `main`.
