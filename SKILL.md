---
name: binary-ninja-plugin-builder
description: Build, debug, and package Binary Ninja (Binja) plugins in Python. Use this skill whenever the user mentions Binary Ninja, Binja, binaryninja, .bntheme-adjacent plugin work, BinaryView, BNDB, LLIL/MLIL/HLIL, PluginCommand, BackgroundTaskThread, sidebar widgets, binaryninjaui, plugin.json, or wants to automate, extend, or script reverse-engineering workflows in Binary Ninja, even if they only say "make a Binja plugin" or "script this in Binary Ninja". Covers command plugins, background tasks, settings, UI/sidebar panels, headless scripts, AI-assisted plugins, testing, and Plugin Manager packaging.
---

# Binary Ninja Plugin Builder

Build working Binja plugins fast, without hallucinated APIs.

## Golden rule: verify the API

Binja's API changes between versions (UI classes especially). Never write an API call from memory without a check when you're not certain it exists. Verify by one of:

- Introspect in the user's install: `python -c "import binaryninja as b; print(b.core_version()); help(b.BinaryView.define_user_symbol)"` or `dir(...)`.
- Grep local examples: Binja ships Python examples, and Vector35/binaryninja-api has `python/examples/` plus `python/*.py` (the source of truth for signatures).
- Docs: docs.binary.ninja (guides, "Developer Guide > Plugins") and api.binary.ninja (Python API reference).
- Real-world patterns: Vector35/community-plugins and popular open-source plugins.

If you can't verify, say so in the code comment and in your reply.

## 0. Scope the plugin (ask only what you can't infer)

- Form: one-off script, menu command, background analysis, sidebar/UI panel, analysis Workflow, or headless tool?
- Target: Binja version/edition, OS, and whether the UI is available (Personal/Non-commercial editions lack headless).
- Inputs: whole binary, a function, an address, a selection, or a range?
- Outputs: comments, renamed symbols, types, tags, a report, or a UI view?
- Distribution: private use, or Plugin Manager release?

## 1. Layout and install location

```
my_plugin/
├── __init__.py      # registers everything at import time
├── plugin.json      # Plugin Manager metadata
├── README.md
└── LICENSE
```

User plugin folder (put the plugin folder inside):

- macOS: `~/Library/Application Support/Binary Ninja/plugins/`
- Linux: `~/.binaryninja/plugins/`
- Windows: `%APPDATA%\Binary Ninja\plugins\`

Use "Plugins > Open Plugin Folder" or the Settings/Log to confirm. Python edits generally need a Binja restart to reload; for fast iteration, develop logic in a normal module and test it in the Python console or headless, and keep `__init__.py` thin.

## 2. Command plugin (the default starting point)

```python
from binaryninja import PluginCommand, log_info, log_error

def run_on_view(bv):
    log_info(f"Analyzing {bv.file.filename}")

def run_on_function(bv, func):
    log_info(f"Function: {func.name} @ {func.start:#x}")

def run_on_address(bv, addr):
    ...

def is_valid_for_function(bv, func):
    return func is not None

PluginCommand.register("MyPlugin\\Run on binary", "Describe what this does", run_on_view)
PluginCommand.register_for_function("MyPlugin\\Run on function", "Describe", run_on_function, is_valid_for_function)
PluginCommand.register_for_address("MyPlugin\\Run at address", "Describe", run_on_address)
```

Variants exist for range and for IL-specific views (e.g. `register_for_high_level_il_function`); check signatures. Use `\\` in the name for submenus. Always provide `is_valid` for context-sensitive commands.

## 3. Long-running work: never block the UI thread

```python
from binaryninja import BackgroundTaskThread, log_error

class MyTask(BackgroundTaskThread):
    def __init__(self, bv):
        super().__init__("MyPlugin: working...", can_cancel=True)
        self.bv = bv

    def run(self):
        try:
            for i, func in enumerate(self.bv.functions):
                if self.cancelled:
                    break
                self.progress = f"MyPlugin: {i}/{len(self.bv.functions)}"
                # ... work ...
        except Exception as e:
            log_error(f"MyPlugin failed: {e}")
        finally:
            self.finish()   # ALWAYS call finish()

PluginCommand.register("MyPlugin\\Start", "Run task", lambda bv: MyTask(bv).start())
```

UI updates and Qt objects must happen on the main thread: use `binaryninja.mainthread.execute_on_main_thread(fn)` (or `..._and_wait`) from worker threads.

## 4. Core API cheat sheet (verify before using)

```python
bv.functions                       # all functions
bv.get_function_at(addr)           # Function or None
bv.get_functions_by_name("main")
bv.get_symbol_by_raw_name(name)
bv.read(addr, length)              # bytes
bv.get_strings()                   # string references
bv.get_code_refs(addr); bv.get_data_refs(addr)
bv.arch, bv.platform, bv.address_size, bv.start, bv.end
func.name = "new_name"             # user-level rename
func.set_comment_at(addr, "text"); bv.set_comment_at(addr, "text")
bv.define_user_symbol(Symbol(...))
bv.define_user_type(name, type)     # parse with bv.parse_type_string("struct X {...}")
func.llil / func.mlil / func.hlil  # IL; may need analysis done; use .ssa_form for dataflow
bv.update_analysis_and_wait()
bv.store_metadata(key, value); bv.query_metadata(key)   # persist data in the BNDB
```

Iterating HLIL:

```python
for block in func.hlil:
    for instr in block:
        if instr.operation == HighLevelILOperation.HLIL_CALL:
            ...
```

Prefer IL over raw disassembly so plugins work across architectures. Never assume pointer size; use `bv.address_size`.

## 5. Settings

```python
import json
from binaryninja import Settings

s = Settings()
s.register_group("myplugin", "My Plugin")
s.register_setting("myplugin.endpoint", json.dumps({
    "title": "Endpoint URL", "type": "string",
    "default": "http://localhost:1234/v1",
    "description": "Where requests are sent."
}))
value = s.get_string("myplugin.endpoint")
```

Never log secrets such as API keys. Never hardcode them.

## 6. UI / sidebar panels

Guard UI code so headless import doesn't crash:

```python
from binaryninja import core_ui_enabled
if core_ui_enabled():
    from binaryninjaui import SidebarWidget, SidebarWidgetType, Sidebar
    from PySide6.QtWidgets import QVBoxLayout, QLabel
    from PySide6.QtGui import QImage
```

Binja bundles PySide6 (older builds used PySide2); import what the installed version provides and verify.

Pattern: subclass `SidebarWidget` (build layout in `__init__`, react in `notifyViewChanged` / `notifyOffsetChanged`), subclass `SidebarWidgetType` with an icon `QImage` and `createWidget(self, frame, data)`, then `Sidebar.addSidebarWidgetType(MyWidgetType())`.

Copy structure from a shipped example or open-source sidebar plugin rather than inventing it; UI class signatures change. Style matches the user's theme automatically; avoid hardcoded colors unless asked.

## 7. Headless scripts

```python
import binaryninja
with binaryninja.load("/path/to/binary") as bv:   # analyzes by default
    for f in bv.functions:
        print(f.name)
```

Requires a license that allows headless use. Use this to unit-test analysis logic on small fixture binaries without launching the UI.

## 8. Plugins that call an LLM or external service

- Do network I/O in a `BackgroundTaskThread`, support cancel, set timeouts, and surface errors via `log_error` and a UI message, not silent failure.
- Make the endpoint, model, and key configurable in Settings (OpenAI-compatible base URL is a good default).
- Send the minimum context (a function's HLIL, not the whole binary) and tell the user what leaves the machine.
- Treat model output as untrusted: parse defensively, validate names and types before applying, and never `eval`/`exec` it.
- Apply changes inside an undo group: `bv.begin_undo_actions()` ... `bv.commit_undo_actions()`, so the user can revert in one step.
- Cap context size and cache results (keyed by function hash/address) to save tokens and time.

## 9. Pitfalls checklist

- Forgetting `self.finish()` leaves a spinner stuck forever.
- Touching Qt/UI from a worker thread crashes or corrupts the UI.
- Mutating `bv.functions` or IL while iterating it; snapshot to a list first.
- Reading IL before analysis finishes; call `update_analysis_and_wait()` in headless contexts (not from inside an analysis callback).
- Using non-user APIs for changes meant to persist in the BNDB; renames, comments, and types should be user-level.
- Assuming one architecture, bit width, or endianness.
- Swallowing exceptions; always log with context.
- Importing UI modules at top level in code that must also run headless.
- Third-party pip dependencies not available in Binja's Python; declare them in `plugin.json` and document install steps.

## 10. Packaging for the Plugin Manager

`plugin.json` (verify fields against the current Plugin Manager docs):

```json
{
  "pluginmetadataversion": 2,
  "name": "My Plugin",
  "author": "Your Name",
  "type": ["helper"],
  "api": ["python3"],
  "description": "One-line summary (what it does).",
  "longdescription": "Longer markdown description shown in the manager.",
  "license": { "name": "MIT", "text": "Full license text here" },
  "platforms": ["Darwin", "Linux", "Windows"],
  "installinstructions": { "Darwin": "pip deps, if any", "Linux": "", "Windows": "" },
  "dependencies": { "pip": [] },
  "version": "1.0.0",
  "minimumbinaryninjaversion": 0
}
```

Set `type` to what applies (e.g. `core`, `ui`, `architecture`, `binaryview`, `helper`), set `minimumbinaryninjaversion` to a real build number you tested, and include a README with screenshots. Community submissions go through Vector35/community-plugins.

## 11. Delivery format

When you hand code to the user:

- Give the full file tree and each file's contents (or write the files to disk).
- State the install path for their OS and the steps to load it.
- List what you verified (ran headless, checked signature) versus what is untested (UI behavior, specific Binja versions).
- Note the minimum Binja version you assumed.
