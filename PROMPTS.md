# CodeWiki — Build Prompts

Four prompts, in order. Each produces something independently demoable.

Prompt 1 writes `docs/DESIGN.md`. Prompts 2-4 begin by reading it, so you never
retype context.

| # | Builds | Needs Copilot? | Demo after |
|---|---|---|---|
| 1 | Python engine — index + skeleton wiki | No | Override matrix, dependency map, symbol index |
| 2 | Extension + chat agent with tools | Yes | `@bofa-codewiki` answering with citations |
| 3 | Prose generation — digests + narratives | Yes, heavily | Readable pages with gotchas |
| 4 | Webview panel | No | Browsable wiki, diagrams, click-through |

Two constraints to keep verbatim in every prompt, because an assistant will
otherwise silently default to the more common pattern:

- **stdlib only, no npm, no TypeScript**
- **`ast.get_docstring`, never scan comments above the `def`**

---

## PROMPT 1 — Python engine

```
Build a Python package `codewiki` in this folder. Stdlib only — no third-party
dependencies. It generates a markdown wiki for a Python repo with zero LLM calls.
Must be repository-agnostic: no hardcoded paths, directory names, or domain terms.

CLI: `python -m codewiki index <repo>` writes to `<repo>/.codewiki/`.

scanner.py — walk the repo; skip .venv, site-packages, __pycache__, build, dist,
migrations, *_pb2.py, and any glob supplied in config; skip files over 400 KB.
Record path, non-blank line count, and sha256 truncated to 16 chars.

ast_extract.py — parse each file with `ast`. Per class and function record: name,
kind, start and end line, signature, decorators, parameter annotations, and
docstring via ast.get_docstring. Do NOT scan comment lines above the `def` —
Python docstrings are the first statement inside the body, and scanning upward
captures decorators instead and misses every docstring in the repo. Build an
import table mapping local name to qualified name, handling `import x`,
`import x as y`, `from x import y`, `from x import y as z`, and relative imports
resolved against the package. Record syntax errors without aborting the run.

hierarchy.py — build the class graph. Resolve each base class name through that
file's import table into a qualname. First parse __init__.py re-exports (so
`from .tarn import TarnPricer` inside insts/__init__.py means insts.TarnPricer ->
insts.tarn.TarnPricer) and apply that map before resolution. Fall back to matching
a base by bare class name ONLY when exactly one class in the entire repo has that
name — otherwise unrelated classes in different packages get fused into one bogus
hierarchy. Detect abstract methods two ways: the @abstractmethod decorator, and a
body whose only statement raises NotImplementedError. For each base with 4 or more
transitive descendants, compute an override matrix: for each descendant, which of
the base's methods it defines (with line number), which it inherits, and which it
leaves abstract.

clustering.py — group files by directory. A directory with 5 or more files whose
classes mostly share one base becomes a "catalog" module; others are plain modules.
Roll directories under 3 files into their parent. Never split a directory
alphabetically.

render.py — write .codewiki/wiki/*.md:
  - an overview page with stats and a mermaid graph of module dependencies
  - a page per module listing files with signatures and first docstring lines
  - a catalog page per hierarchy with the override matrix as a table, naming
    outliers explicitly: which subclasses override an unusual method, and which
    leave one unimplemented
Also write .codewiki/index/{files,symbols,hierarchy,modules}.json.

Every statement referring to code carries a [path:line] citation.

Add pytest tests for ast_extract and hierarchy, including these two cases:
  - a class inheriting through an __init__.py re-export resolves correctly
  - two same-named classes in different packages are NOT merged

Finally write docs/DESIGN.md recording the architecture, the file formats, and the
reasoning behind these decisions.
```

---

## PROMPT 2 — Extension and chat agent

```
Read docs/DESIGN.md first.

PART A — add a `tool` subcommand to the engine:
`python -m codewiki tool <name> --repo <path> --json '<args>'`, printing JSON to
stdout. Tools:

  find_symbol(name)            matches from symbols.json with kind, line range,
                               signature, docstring
  find_implementations(base)   transitive subclasses from hierarchy.json, each with
                               path, line, and which methods it overrides
  find_overrides(base, method) who overrides it, who inherits, who leaves it abstract
  search_wiki(query)           BM25 over wiki pages, symbols and docstrings;
                               tokenizer splits identifiers on underscores and
                               camelCase; boost exact identifier matches
  read_file(path, start, end)  line-numbered output
  grep(pattern, glob)          over the scanned file set only, not the whole
                               workspace, so results stay consistent with the wiki

PART B — create a VS Code extension in a new folder `codewiki-ext`. Plain CommonJS
JavaScript only — no TypeScript, no npm install, no compile step, since npm is
unavailable. Use require('vscode').

package.json: main "./extension.js", engines.vscode "^1.95.0", activationEvents
["onStartupFinished"], a chatParticipants entry with id "codewiki.agent" and name
"bofa-codewiki", and commands codewiki.build and codewiki.status. Add
.vscode/launch.json with a configuration of "type": "extensionHost".

extension.js:
- Resolve the Python interpreter from the ms-python.python extension's active
  environment path, falling back to "python". Log all activity to an output channel.
- codewiki.build runs `python -m codewiki index <workspaceFolder>` via
  child_process.execFile, with a progress notification.
- If workspaceFolders has more than one entry, prompt for which to use rather than
  silently taking the first.
- Register the chat participant. In the handler use request.model — do NOT call
  selectChatModels. Declare all six engine tools plus find_references.
- Implement find_references with
  vscode.commands.executeCommand('vscode.executeReferenceProvider', uri, position).
  Implement every other tool by shelling out to the engine.
- Loop up to 10 iterations: send the request with tools; collect
  LanguageModelToolCallPart objects from the stream; execute them; send results back
  as a LanguageModelChatMessage.User containing LanguageModelToolResultPart objects
  paired one-to-one with the calls. Do NOT stringify tool results — that breaks
  multi-hop reasoning and can cause the request to be rejected.
- Stream a progress line per tool call. Rewrite [path:line] in the final answer into
  markdown links that open the file at that line. Emit stream.reference() for each
  file the agent read.
- System prompt: start from search_wiki for orientation; verify claims against files
  actually read this turn, because the wiki can lag the code; cite [path:line]
  inline on the specific claim rather than bundled at the end; state plainly when
  something is not found or when two implementations exist and it is unclear which
  is live; never invent a path, symbol, or line number.

The extension must be repository-agnostic — no hardcoded paths or domain terms.
Update docs/DESIGN.md.
```

---

## PROMPT 3 — Prose generation

Only after using prompt 2 for a week, and scope it deliberately. Prompt 2 answers
targeted questions well on its own; prose adds fuzzy retrieval, architecture
synthesis, and browsable documentation.

```
Read docs/DESIGN.md first.

Add LLM-backed prose generation to the engine, driven from the extension.

queue.py — jobs with kind (digest | digest_batch | page | overview), key, state,
attempts, priority. Persist .codewiki/queue.json after EVERY completed job, not in
batches. Order by priority descending. Concurrency 1 with ~1s spacing. On a throttle
error, double the delay and requeue; give up after 5 attempts. On quota exhaustion,
stop cleanly and allow resume. Page jobs become eligible only when every digest in
that module is done; the overview only when every page is done — an overview
generated from a partial set describes an architecture that does not exist.

Priority: log1p(subclass_count)*3 + log1p(inbound_imports)*2 + log1p(loc)*0.5, plus
a bonus for entry points and for test files (in Python, tests document intended
behaviour more reliably than comments).

Batching: pack files under 150 lines, 8 at a time, into a single call returning a
JSON array keyed by path. This typically removes half the calls.

Truncation: never slice source at a character offset — that hands the model a
syntactically broken file. Always include every signature, docstring, decorator and
module-level constant, then fill the remaining token budget with full function
bodies in descending size order, replacing the rest with "...".

Subclass digests: when a file's main class subclasses a known base, include the
base's abstract method signatures in the prompt and ask only what this
implementation does DIFFERENTLY — its conventions, deviations, and assumptions. Do
not let forty subclasses each re-explain the base contract.

Prompts live in one module. Every prompt requires [path:start-end] citations on each
factual claim and instructs the model to omit any claim it cannot support rather
than infer it. The digest returns strict JSON with fields purpose, key_symbols,
notes, depends_on — where notes covers invariants, ordering requirements, units and
conventions, and silent failure modes, skipping anything visible at a glance.

Extension side: add codewiki.buildProse and codewiki.resume. This channel uses
vscode.lm.selectChatModels({vendor:'copilot'}), NOT request.model. Enumerate the
available families and pick a main model and a cheaper one by preference order — do
not hardcode model names, they are not API model strings. Use model.maxInputTokens
and await model.countTokens() for budgeting. Handle LanguageModelError:
NoPermissions means stop and do not retry; quota or rate messages mean back off and
requeue. Show a status bar item with the remaining job count.

Add --limit-to <glob> so a first run can cover one directory only. Digests are keyed
by content hash and skipped when unchanged. Sweep orphaned digests for deleted files
at the end of each build.

Update docs/DESIGN.md.
```

---

## PROMPT 4 — Wiki panel

```
Read docs/DESIGN.md first.

Add a codewiki.open command that renders the wiki in a createWebviewPanel.

- Vendor mermaid and marked into media/vendor/ — do NOT load them from a CDN,
  because the network may be proxied and the panel would silently degrade to
  unrendered markdown with no diagrams. Set localResourceRoots, build URIs with
  webview.asWebviewUri, and scope the CSP to webview.cspSource.
- Sidebar listing all pages; main area renders the markdown.
- Rewrite [path:line] into clickable spans BEFORE markdown parsing, so the links
  survive into the rendered output. Clicking posts a message; the extension opens
  the document in ViewColumn.Two and reveals that line.
- Render mermaid code blocks as diagrams.
- Style entirely from VS Code theme variables (--vscode-*), so the panel inherits
  the editor theme rather than fighting it.
- Show a staleness banner when any file hash in the page's input_hashes differs from
  the current index.
- One panel only, reused across navigation; the title carries the page name.

Update docs/DESIGN.md.
```

---

## After the build

Verify the tool is repository-agnostic — this is what makes it reusable without
re-prompting:

```
grep -ri "insts\|TARN\|SABR\|<your-repo-name>" codewiki-ext/ codewiki/
```

Any hit outside a test fixture is a bug: something repo-specific leaked into the
tool, and that is the only thing that would force re-prompting for a second repo.

Onboarding a further repo then requires no LLM and no code change:

1. Open it in VS Code — the extension is already installed
2. Run **CodeWiki: Build wiki**
3. Adjust `codewiki.exclude` if the output shows generated or vendored code

To change the tool itself later, the prompt is always "read docs/DESIGN.md, then
make this change" — a diff against existing code, never a rebuild.
