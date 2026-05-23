# Building a regex debugger for the terminal in Rust

*Cross-post from: https://github.com/brevity1swos/rgx*

---

I built [rgx](https://github.com/brevity1swos/rgx), a terminal regex debugger written in Rust. The v0.12.3 release is out today. Here's what's in it and why I built it the way I did.

## The problem

I write regex-heavy code in terminal-centric workflows — SSH sessions, containers, scripts that pipe data around. Every time I needed to debug a pattern, I'd break flow to open a browser, tab over to regex101.com, paste in the pattern and test string, iterate, copy the pattern back, close the tab.

The gap isn't just convenience. regex101.com uses PCRE2, JavaScript, or Python regex engines. When you're writing Rust code, you're running the `regex` crate — which has no lookaround, different Unicode semantics, and different performance characteristics. Testing in the wrong engine gives you false confidence.

I wanted something that worked in the terminal, stayed in my workflow, and ran against the actual engines my code uses.

## What rgx does

The core loop is fast: type a pattern, see matches update in real time with capture group highlighting and a plain-English explanation.

```bash
rgx '\d{3}-\d{3}-\d{4}'                    # start with a pattern
echo "Call 555-123-4567" | rgx '\d+'        # pipe stdin as test string
rgx -p -t "error 404" '\d+' | sort          # batch mode in a pipeline
```

Beyond the basics:

### Step-through debugger (Ctrl+D, PCRE2)

This is the part I haven't seen elsewhere. Pressing Ctrl+D opens a dual-cursor trace over the PCRE2 engine's execution — one cursor on the pattern, one on the test string, moving together as the engine works through the match. Backtracking steps are marked distinctly. There's a heatmap mode (H) that shows which parts of the pattern were touched most often, which makes catastrophic backtracking obvious at a glance.

It's the difference between "the pattern doesn't match, I don't know why" and "here is exactly where the engine gave up."

### Three engines with auto-selection

- **Rust `regex`** (default) — linear time, no lookaround, no backreferences
- **fancy-regex** — adds lookaround and backreferences, pure Rust
- **PCRE2** — adds possessive quantifiers, recursion, conditionals

When you type a pattern that uses lookahead, rgx upgrades automatically to the simplest engine that supports it. The status bar shows the current engine. You can override with Ctrl+E or `--engine`.

### Code generation (Ctrl+G)

Once a pattern works, Ctrl+G generates ready-to-paste code for 8 languages: Rust, Python, JavaScript, Go, Java, C#, PHP, Ruby. It handles the escaping, the compile call, the match loop — everything you'd have to look up.

### Generate regex from examples (Ctrl+X)

Ctrl+X opens a [grex](https://crates.io/crates/grex) overlay. Type example strings, one per line, and grex generates a pattern that matches all of them. Tab loads it into the main editor. Useful for formats you don't have the regex memorized for.

### Live filter mode

```bash
cat app.jsonl | rgx filter --json '.msg' '(?i)error'
git diff | rgx filter '^\+.*console\.log'
```

`rgx filter` reads stdin or a file and lets you refine the pattern in a TUI. When you press Enter, matching lines go to stdout. Non-TTY stdout skips the TUI entirely, so it composes in pipelines. The `--json <PATH>` flag extracts a specific field from JSONL records before matching — useful when you want to filter by message content without the pattern accidentally matching timestamps or IDs in the raw line.

### Test suite mode

```toml
# tests/urls.toml
pattern = "https?://[^\\s]+"
engine = "rust"

[[tests]]
input = "visit https://example.com today"
should_match = true
```

```bash
rgx --test tests/urls.toml   # exit 0 = all pass, 1 = failures, 2 = error
```

Regex regressions are easy to introduce and hard to catch without CI assertions. This makes it easy to keep a test suite next to the patterns in your repo.

## Implementation notes

rgx is built on [ratatui](https://ratatui.rs/) + crossterm for the TUI, with arboard for clipboard, grex for example-to-regex generation, and direct PCRE2 FFI for the step-through debugger.

The step-through debugger uses PCRE2's callout mechanism — a callback that PCRE2 invokes at each match step. The callback records a `DebugStep` (pattern offset, subject offset, backtrack flag) into a `Vec`, and the UI replays those steps as you navigate with arrow keys. The heatmap is a simple frequency count over pattern offsets, rendered as a color gradient.

Auto engine selection works by walking the regex AST before compilation: if the pattern uses lookahead, backreferences, or PCRE2-specific syntax, `detect_minimum_engine()` returns the smallest engine that supports those features. The engine only upgrades, never downgrades mid-session.

The filter subsystem (`rgx filter`) is intentionally separate from the main TUI state — different UX, different lifecycle. Non-TTY detection happens at startup: if stdout isn't a terminal, the TUI never launches and matching lines go directly to stdout, which keeps the piping semantics clean.

## Install

```bash
cargo install rgx-cli                    # crates.io (pure Rust, no PCRE2)
cargo install rgx-cli --features pcre2-engine  # with step-through debugger
brew install brevity1swos/tap/rgx        # Homebrew
yay -S rgx-cli                           # AUR
```

GitHub: https://github.com/brevity1swos/rgx

---

*Questions or issues: https://github.com/brevity1swos/rgx/issues*
