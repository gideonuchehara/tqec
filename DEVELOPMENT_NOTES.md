# TQEC Development Setup & Debugging Guide

## 1. Installing `uv`

`uv` is a fast Python package and environment manager (written in Rust) that replaces `pip` + `venv` + `pyenv`. It is the primary tool used by this project.

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc
uv --version  # verify
```

---

## 2. Setting Up the Project Environment

Navigate to the project root and run these steps **once**:

### Create the virtual environment
```bash
cd ~/projects/tqec
uv venv
```
This creates a `.venv/` folder — an isolated Python sandbox for the project.

### Install the project and all dev dependencies
```bash
uv sync --group dev
```
This installs `tqec` in editable mode (changes to `src/tqec/` take effect immediately) plus all development tools (`pytest`, `ruff`, etc.).

> **Why `uv sync` and not `uv pip install`?**
> This project uses `[dependency-groups]` in `pyproject.toml` (a `uv`-specific feature).
> `uv sync` reads the full spec and ensures everything matches exactly.

### Activate the environment
```bash
source .venv/bin/activate
```
Your prompt will change to show `(.venv)`. Now `python` and `pytest` refer to the project's versions, not the system ones.

### Verify the install
```bash
python --version && which python   # should show .venv/bin/python
python -c "import tqec; print(tqec.__version__)"  # should print 0.2.0
```

---

## 3. Running Tests

```bash
# Run all tests (quick check)
uv run pytest tests/ -x -q

# Run a specific test file
pytest tests/compile/compile_test.py -s

# List all test cases in a file (without running)
pytest tests/compile/compile_test.py --collect-only -q

# Run one specific parametrized test case
pytest "tests/compile/compile_test.py::test_compile_memory[1-convention0-ZXZ]" -s
```

### Common flags

| Flag | Meaning |
|------|---------|
| `-x` | Stop at first failure |
| `-q` | Quiet output |
| `-s` | Don't capture stdout/stdin (required for `pdb`) |
| `-k "keyword"` | Filter tests by keyword |
| `--collect-only` | List tests without running them |

---

## 4. Python Debugging with `pdb`

`pdb` is Python's built-in command-line debugger — the equivalent of `gdb` for C++.

### Step 1: Add a breakpoint

Insert `breakpoint()` anywhere in the source code where you want execution to pause:

```python
def test_compile_memory(...):
    g = BlockGraph("Memory Experiment")
    breakpoint()  # execution pauses here
    g.add_cube(Position3D(0, 0, 0), kind)
```

> `breakpoint()` is the modern Python 3.7+ way. It respects the `PYTHONBREAKPOINT`
> environment variable, allowing you to swap debuggers without changing code.

### Step 2: Run the test with `-s`

```bash
pytest "tests/compile/compile_test.py::test_compile_memory[1-convention0-ZXZ]" -s
```

The `-s` flag is **required** — without it pytest captures stdin and pdb cannot receive your commands.

### Step 3: Use the debugger

When execution hits `breakpoint()`, you will see:
```
> /path/to/file.py(171)test_compile_memory()
-> g.add_cube(Position3D(0, 0, 0), kind)
(Pdb)
```
- The line after `>` is where you are paused
- `->` is the **next line about to execute** (has not run yet)
- `(Pdb)` is the prompt waiting for your command

---

## 5. `pdb` Command Reference

| Command | Action | `gdb` equivalent |
|---------|--------|-----------------|
| `p x` | Print variable `x` | `print x` |
| `ll` | Show current function source with arrow at current line | `list` |
| `n` | Step over (execute line, stay at same level) | `next` |
| `s` | Step into (follow the call inside the function) | `step` |
| `r` | Run until current function returns | `finish` |
| `c` | Continue until next breakpoint or end | `continue` |
| `bt` | Show full call stack | `backtrace` |
| `u` | Move up one frame in the call stack | `up` |
| `d` | Move down one frame in the call stack | `down` |
| `q` | Quit the debugger | `quit` |

### Reading the call stack (`bt`)

```
pytest binary              ← entry point
  → pytest internals       ← framework plumbing (ignore these)
    → test_compile_memory  ← your test code
      → generate_circuit_and_assert  ← where you are now
```

Read bottom to top. The `>` marks your current frame. Use `u`/`d` to navigate frames and inspect variables at each level without restarting.

---

## 6. Cleanup

Always remove `breakpoint()` before committing:

```bash
git diff  # check you haven't left any breakpoints in
```

Or use `grep` to find any stray breakpoints:

```bash
grep -rn "breakpoint()" tests/ src/
```

---

## 7. `uv` Quick Reference

| Command | Purpose |
|---------|---------|
| `uv venv` | Create `.venv/` virtual environment |
| `uv sync` | Install project + default dependencies |
| `uv sync --group dev` | Install project + dev group |
| `uv sync --group all` | Install everything (dev, docs, bench) |
| `uv pip install <pkg>` | Install a package into the active venv |
| `uv run <cmd>` | Run a command inside the venv without activating |
| `uv python list` | List available Python versions |
| `uv python install 3.12` | Install a specific Python version |

---

## 8. Pauli Webs and Correlation Surfaces

### What is a Pauli Web?

A **Pauli web** (from the pyzx library) labels each **half-edge** of a ZX graph with a Pauli operator (X, Y, or Z). The "RGB" naming in the pyzx demo notebook maps R=X, G=Z, B=Y. A Pauli web is "closed" when every spider's adjacent half-edges form a stabilizer of that spider.

Reference notebook: https://nbviewer.org/github/zxcalc/pyzx/blob/master/demos/PauliWebsRGB.ipynb

### How it relates to tqec's CorrelationSurface

tqec's `CorrelationSurface` is the same concept, embedded in 3D topological quantum error correction. Instead of coloring half-edges on an abstract ZX diagram, it tracks how logical Pauli operators flow through a quantum error correction circuit topology.

The interop layer in `src/tqec/interop/pyzx/correlation.py` converts between the two representations.

### The single-node round-trip bug (PR #926)

pyzx's `PauliWeb` stores half-edges — `(v, w)` is the end of an edge nearest to `v`. This **requires an actual edge to exist**. A single isolated node (no edges) has no half-edges, so pyzx has nowhere to store the Pauli label.

tqec's `CorrelationSurface` *does* support single-node surfaces (stored as a self-loop `ZXEdge(node, node)`), which exposed a gap when converting to/from `PauliWeb`.

**Three chained bugs were fixed:**

1. `_graph_view` in `correlation.py` duplicated self-loop edges (u==v inserted twice), causing XOR cancellation: `Pauli.Z ^ Pauli.Z = Pauli.I`.
2. `_to_mutable_graph_representation` called `is_hadamard(zx_graph, (u, u))` on a ghost edge that doesn't exist in the ZX graph.
3. pyzx silently drops self-loop half-edges — `pauli_web.half_edges()` returns `{}` for single-node graphs.

**Fix strategy:** detect the single-node case upfront in `pauli_web_to_correlation_surface` and reconstruct the surface directly from the vertex type, bypassing `PauliWeb` entirely.

### X↔Z basis flip (important)

The correlation surface basis is the **opposite** of the spider's own type:
- X-type spider → `Basis.Z` surface
- Z-type spider → `Basis.X` surface

This is because a single X spider is stabilized by Z operators on its legs, and vice versa. Always pass `zx_graph.phase(u)` alongside `zx_graph.type(u)` to `vertex_type_to_pauli` so non-zero phase vertices (e.g. S node with phase=1/2 → `Pauli.Y`) are handled correctly.
