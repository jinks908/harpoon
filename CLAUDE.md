# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**IMPORTANT GUIDELINES**

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```
## 5. Check for Existing References First

**Avoid re-reading the same file(s) multiple times.**

- Before pulling a file into the conversation from scratch with a `Read` call, first check
  if a copy already exists in the conversation/context. If it does, reference that copy
  instead of re-reading the file.
- Only pull a fresh file if that file has been modified since it was last read.

---

# Repository Notes

This is Harpoon 2 (the `harpoon2` rewrite), a Neovim plugin written in Lua. `plenary.nvim` is
the only runtime dependency; `telescope.nvim` is optional and only needed for the Telescope
extension.

## Commands

```bash
make test        # run the full test suite (headless nvim + PlenaryBustedDirectory)
make fmt         # stylua lua/ --config-path=.stylua.toml
make lint        # luacheck lua/ --globals vim
make pr-ready    # fmt + lint + test — run this before opening a PR
```

Run a single spec file (PlenaryBustedDirectory takes a directory; use `PlenaryBustedFile` for
one file):

```bash
nvim --headless --noplugin -u scripts/tests/minimal.vim \
  -c "PlenaryBustedFile lua/harpoon/test/list_spec.lua"
```

`scripts/tests/minimal.vim` puts `.` and `../plenary.nvim` on the runtimepath, so **plenary must
be checked out as a sibling directory of this repo** for the tests to run locally. CI instead
symlinks both into `~/.local/share/nvim/site/pack/vendor/start`.

CI (`.github/workflows/`) runs tests against nvim nightly and v0.9.0, plus `stylua --check` and
`luacheck` as separate required jobs — formatting failures fail the build.

## Architecture

The core is four layers, each of which only knows about the one below it:

- **`init.lua`** — returns a *singleton* `Harpoon` instance (`local the_harpoon = Harpoon:new()`
  at module load), not a class. `Harpoon.setup()` handles being called as both `harpoon:setup(cfg)`
  and `harpoon.setup(cfg)` by detecting whether `self` is the singleton. Owns the `lists` cache,
  keyed `lists[key][name]` where `key` is `config.settings.key()` (cwd by default) — this is what
  makes lists project-scoped.
- **`list.lua`** — `HarpoonList`, the in-memory item list. `config.lua` — default config plus the
  per-list config merge (`get_config` overlays `config[name]` on `config.default`).
- **`data.lua`** — persistence. Writes one JSON file per key into `stdpath("data")/harpoon/`,
  named by `sha256(key)`. `Data:sync()` re-reads the file and merges before writing, so
  concurrent nvim instances don't clobber each other's keys.
- **`ui.lua` / `buffer.lua`** — the floating quick menu. `ui.lua` owns the window; `buffer.lua`
  owns the menu buffer's keymaps (`q`, `<Esc>`, `<CR>`) and autocmds. The menu is an *editable
  buffer*: `BufWriteCmd` calls `ui:save()`, which reads the buffer lines back and hands them to
  `list:resolve_displayed()` to diff against the current items.

### The sparse-array invariant (most common source of bugs)

`HarpoonList.items` is a **sparse** array — deleting an item sets `items[i] = nil` rather than
shifting, so indices stay stable and `#items` is unreliable. The list tracks `self._length`
separately, and `guess_length`/`determine_length` in `list.lua` maintain it. Always iterate
`for i = 1, self._length` and use `list:length()`; never `#list.items` or `ipairs`.

### Everything is configurable via the item lifecycle

A list doesn't know it holds files. `config.default` supplies `create_list_item`, `display`,
`select`, `equals`, `encode`, `decode`, and `get_root_dir`; overriding these under a named key in
`setup()` makes that list hold terminals, commands, or anything else. `encode = false` opts a
list out of persistence entirely (checked in `Harpoon:sync()`).

### Extensions / events

`extensions/init.lua` is a global event bus (`Extensions.extensions` is a singleton, so listeners
persist across reloads — tests call `clear_listeners()`). Event names are in `event_names`.
Notably, `init.lua` registers its own listener at construction so that `ADD`/`REMOVE`/`REORDER`/
`LIST_CHANGE`/`POSITION_UPDATED` each trigger a `harpoon:sync()` — **mutations persist by
emitting the right event**, not by calling sync directly. The UI emits a single `LIST_CHANGE`
instead of a burst of add/remove events.

### Tests

`lua/harpoon/test/utils.lua` provides the harness. `utils.before_each(os.tmpname())` overrides
`Data.test.set_fullpath` to redirect persistence to a temp file, reloads the `harpoon` module
(the singleton means state leaks between tests otherwise), and sets `settings.key` to the
constant `"testies"`. Tests that touch real buffers use `create_file`/`clean_files` and the
checkpoint-file mechanism to get back to a known buffer.

