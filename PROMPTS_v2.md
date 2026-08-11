# CodeWiki — Build Prompts

Paste **CONTEXT** once at the start of a session, then one stage prompt at a time.
Do not paste all four stages up front — long context makes assistants start later
work early and produce shallow versions of everything.

Prompt 1 writes `docs/DESIGN.md`. Prompts 2-4 begin by reading it, so you never
retype context after the first session.

| # | Builds | Needs Copilot? | Demo after |
|---|---|---|---|
| 1 | Python engine — index, hierarchy, catalog pages | No | Override matrix, dependency map, symbol index |
| 2 | Extension + chat agent with tools | Yes | `@bofa-codewiki` answering with citations |
| 3 | Prose generation — digests + narratives | Yes, heavily | Readable pages with gotchas |
| 4 | Webview panel | No | Browsable wiki, diagrams, click-through |

**Four lines to keep verbatim wherever they appear**, because an assistant will
otherwise silently default to the opposite:

- stdlib only, no npm, no TypeScript
- `ast.get_docstring`, never scan comments above the `def`
- prompt 1 is deterministic — no LLM calls anywhere in it
- tool results use `LanguageModelToolResultPart`, never stringified

---

## CONTEXT — paste once at the start of each session

```
I'm building CodeWiki: a VS Code extension that generates a source-linked wiki for
a Python codebase and answers questions about it through a chat agent registered in
GitHub Copilot Chat.

Target codebase: a financial instrument pricing library, roughly 1,000-5,000 Python
files. Dispatch is class-hierarchy based — abstract bases with many concrete
subclasses — and implementations are also grouped by directory (instruments in one,
models in another). There is no decorator-registry dispatch.

Critically: annotation coverage and docstring coverage are both LOW. The codebase
does not describe itself. This means (a) structural facts must be recovered from
the syntax tree rather than read off type hints, (b) inline comments are the main
natural-language content that exists in the source, and (c) LLM-generated prose is
not a nice-to-have — it is the only documentation this code will ever have.

Firm constraints:
- LLM access is GitHub Copilot only, through the vscode.lm API. No API keys, no
  vendor SDKs, no HTTP gateway. Rate limits are undocumented, so call volume is the
  binding resource.
- No git. Version control is an internal system that may leave no filesystem
  marker. Change detection must be content-hash based. Do not use `git log` for
  anything.
- npm may be blocked. The extension is plain CommonJS JavaScript with no build
  step. The Python engine is stdlib-only, no pip dependencies.
- Everything runs locally. No localhost server, no listening socket.
- The tool must be repository-agnostic: no hardcoded paths, directory names, or
  domain terms anywhere in the code. The repo root is a runtime parameter.

Four stages, built in order:
  1. Python engine — deterministic index and structural pages, zero LLM calls
  2. VS Code extension and chat agent with tools
  3. LLM prose generation — per-file digests and page narratives
  4. Webview panel for browsing

Each stage records its decisions in docs/DESIGN.md so later stages don't need
re-explaining. Ask me before making an architectural choice this context doesn't
cover.
```

---

## PROMPT 1 — Python engine (deterministic, no LLM)

```
Build a Python package `codewiki` in this folder. Stdlib only — no third-party
dependencies. Everything in this stage is deterministic AST analysis: do NOT call
any LLM, and do not infer beyond what the syntax tree states.

CLI: `python -m codewiki index <repo>` writes to `<repo>/.codewiki/`.

scanner.py — walk the repo; skip .venv, site-packages, __pycache__, build, dist,
migrations, *_pb2.py, and any glob supplied in config; skip files over 400 KB.
Record path, non-blank line count, and sha256 truncated to 16 chars.

ast_extract.py — parse each file with `ast`. Per class and function record: name,
kind, start and end line, signature, decorators, parameter annotations where
present, and docstring via ast.get_docstring. Do NOT scan comment lines above the
`def` — Python docstrings are the first statement inside the body, and scanning
upward captures decorators instead and misses every docstring in the repo. Build an
import table mapping local name to qualified name, handling `import x`,
`import x as y`, `from x import y`, `from x import y as z`, and relative imports
resolved against the package. Record syntax errors without aborting the run.

Also in ast_extract.py, two additions that matter because this codebase has low
annotation and docstring coverage:

(a) PARAMETER SHAPES. For each function, infer each parameter's required interface
from how the body uses it: which methods are called on it, which attributes are
read, whether it is indexed, whether it is iterated. Store as param_shapes on the
symbol record. In duck-typed code this is the real contract and substitutes for the
missing type hints — e.g. "curve: calls .discount_factor(), .zero_rate()".

(b) INLINE COMMENTS. Extract them with the `tokenize` module, since ast discards
comments. Skip pragma comments (type:, noqa, pylint, fmt:). Attach each comment to
the INNERMOST definition whose line range contains it — a class must not absorb its
methods' comments. Store as leading_comments and inline_comments on the symbol
record. With docstring coverage low, these are the primary natural-language content
in the source.

hierarchy.py — build the class graph. Resolve each base class name through that
file's import table into a qualname. First parse __init__.py re-exports (so
`from .tarn import TarnPricer` inside insts/__init__.py means insts.TarnPricer ->
insts.tarn.TarnPricer) and apply that map before resolution. Fall back to matching a
base by bare class name ONLY when exactly one class in the entire repo has that name
— otherwise unrelated classes in different packages get fused into one bogus
hierarchy. Detect abstract methods two ways: the @abstractmethod decorator, and a
body whose only statement raises NotImplementedError. For each base with 4 or more
transitive descendants, compute an override matrix: for each descendant, which of
the base's methods it defines (with line number), which it inherits, and which it
leaves abstract.

clustering.py — group files by directory. A directory with 5 or more files whose
classes mostly share one base becomes a "catalog" module; others are plain modules.
Roll directories under 3 files into their parent. Never split a directory
alphabetically. When a directory's classes predominantly share one base, title the
page from the base class rather than the folder name.

pairings.py — emit a bipartite graph from the import edges: for each file in one
domain directory, which modules in another domain directory does it import. If the
result is sparse, report that explicitly rather than hiding it — it would mean
pairing is resolved at runtime rather than statically.

render.py — write .codewiki/wiki/*.md and .codewiki/index/*.json.
Priority order, because this repo has little inline documentation to display:
  1. Catalog page per hierarchy — the override matrix as a table, naming outliers
     explicitly: which subclasses override an unusual method, which leave one
     abstract. Above 30 implementations, collapse the table into a summary
     ("38 of 40 override price; 6 override greeks; 3 do not implement schedule")
     and name the outliers. The outliers are the information.
  2. Overview page — stats plus a mermaid graph of module dependencies.
  3. Module pages — keep these minimal. Signature plus param_shapes plus any
     comments. Do not invest effort here; stage 3 supplies the prose.
JSON files: files.json, symbols.json, hierarchy.json, modules.json, pairings.json.

Every statement referring to code carries a [path:line] citation.

Add pytest tests for ast_extract and hierarchy, including these cases:
  - a class inheriting through an __init__.py re-export resolves correctly
  - two same-named classes in different packages are NOT merged
  - a comment inside a method is attributed to the method, not the enclosing class
  - param_shapes on a function using curve.discount_factor() and vol.alpha

Finally write docs/DESIGN.md recording the architecture, the file formats, and the
reasoning behind these decisions — including why AST is used for bulk extraction
rather than the language server.
```

---

## PROMPT 2 — Extension and chat agent

```
Read docs/DESIGN.md first.

PART A — add a `tool` subcommand to the engine:
`python -m codewiki tool <name> --repo <path> --json '<args>'`, printing JSON to
stdout. Tools:

  find_symbol(name)            matches from symbols.json with kind, line range,
                               signature, param_shapes, docstring, comments
  find_implementations(base)   transitive subclasses from hierarchy.json, each with
                               path, line, and which methods it overrides
  find_overrides(base, method) who overrides it, who inherits, who leaves it abstract
  search_wiki(query)           BM25 over wiki pages, symbols, docstrings, extracted
                               comments and test function names; tokenizer splits
                               identifiers on underscores and camelCase; boost exact
                               identifier matches
  read_file(path, start, end)  line-numbered output
  grep(pattern, glob)          over the scanned file set only, not the whole
                               workspace, so results stay consistent with the wiki

Index test function names in search_wiki alongside symbols — with docstring coverage
low, test names carry behavioural vocabulary the implementation identifiers do not.

PART B — create a VS Code extension in a new folder `codewiki-ext`. Plain CommonJS
JavaScript only — no TypeScript, no npm install, no compile step, since npm is
unavailable. Use require('vscode').

package.json: main "./extension.js", engines.vscode "^1.95.0", activationEvents
["onStartupFinished"], a chatParticipants entry with id "codewiki.agent" and name
"bofa-codewiki", and commands codewiki.build, codewiki.selectRepo, codewiki.status.
Add .vscode/launch.json with a configuration of "type": "extensionHost".

REPOSITORY SELECTION. The workspace may contain several repositories, and the wiki
is built for exactly one. Build a candidate list from every entry in
vscode.workspace.workspaceFolders, plus each immediate subdirectory of a workspace
folder that looks like a repo root (contains .git, .svn, CVS, pyproject.toml,
setup.py, or a subdirectory with __init__.py). codewiki.build shows a quick-pick
labelled by folder name with the full path as detail. Persist the choice in
workspaceState so later builds do not re-ask; codewiki.selectRepo changes it. If
only one candidate exists, use it without prompting. Write .codewiki/ into the
SELECTED repo's root, never the workspace root. Show the selected repo in the
status bar.

extension.js:
- Resolve the Python interpreter from the ms-python.python extension's active
  environment path, falling back to "python". Log all activity to an output channel.
- codewiki.build runs `python -m codewiki index <selectedRepo>` via
  child_process.execFile, with a progress notification.
- Register the chat participant. In the handler use request.model — do NOT call
  selectChatModels. Declare all six engine tools plus find_references.
- Implement find_references with
  vscode.commands.executeCommand('vscode.executeReferenceProvider', uri, position).
  Pylance resolves across the whole workspace, so label each result as inside or
  outside the selected repo and group them separately; report out-of-repo hits as
  "also referenced in <other-repo>" rather than mixing them in.
- Implement every other tool by shelling out to the engine.
- Loop up to 10 iterations: send the request with tools; collect
  LanguageModelToolCallPart objects from the stream; execute them; send results back
  as a LanguageModelChatMessage.User containing LanguageModelToolResultPart objects
  paired one-to-one with the calls. Do NOT stringify tool results — that breaks
  multi-hop reasoning and can cause the request to be rejected.
- If the first iteration returns zero tool calls, fall back to a retrieval-only
  answer rather than looping to nothing; not every model in the Copilot picker
  handles tools well. Say which mode produced the answer.
- Stream a progress line per tool call. Rewrite [path:line] in the final answer into
  markdown links that open the file at that line. Emit stream.reference() for each
  file the agent read. Show the active model name in a footer so a poor answer is
  diagnosable.
- System prompt: start from search_wiki for orientation; verify claims against files
  actually read this turn, because the wiki can lag the code; cite [path:line]
  inline on the specific claim rather than bundled at the end; state plainly when
  something is not found, or when two implementations exist and it is unclear which
  is live; never invent a path, symbol, or line number. Note that this codebase has
  few docstrings, so do not assume an absent docstring means a trivial function —
  read the body.

The extension must be repository-agnostic — no hardcoded paths or domain terms.
Update docs/DESIGN.md.
```

---

## PROMPT 3 — Prose generation

Not optional for this codebase. With annotation and docstring coverage both low,
these digests are the only documentation the code will ever have, and the only
natural-language vocabulary the search index can offer.

```
Read docs/DESIGN.md first.

Add LLM-backed prose generation to the engine, driven from the extension.

queue.py — jobs with kind (digest | digest_batch | page | overview), key, state,
attempts, priority. Persist .codewiki/queue.json after EVERY completed job, not in
batches. Order by priority descending. Concurrency 1 with ~1s spacing plus jitter.
On a throttle error, double the delay and requeue; give up after 5 attempts. On
quota exhaustion, stop cleanly and allow resume. Page jobs become eligible only when
every digest in that module is done; the overview only when every page is done — an
overview generated from a partial set describes an architecture that does not exist.

Priority: log1p(subclass_count)*3 + log1p(inbound_imports)*2 + log1p(loc)*0.5, plus
a bonus for entry points and for test files (in Python, tests document intended
behaviour more reliably than comments, especially where docstrings are absent).

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
Include param_shapes and extracted comments in the digest prompt context.

Because this codebase has almost no docstrings, instruct the digest prompt to state
what each parameter must provide, based on how the body uses it. That information
exists nowhere else and is the single most useful thing a digest can capture here.

HUMAN CORRECTIONS MUST SURVIVE REGENERATION. Each module supports an adjacent file
.codewiki/wiki/<page-id>.notes.md which the generator never writes. Its contents are
appended to the rendered page under a "Notes from the team" heading, and are
included in the prompt context when that page is regenerated so the model does not
contradict them. With no docstrings to cross-check generated prose against, this is
how corrections get made and kept.

Extension side: add codewiki.buildProse and codewiki.resume. This channel uses
vscode.lm.selectChatModels({vendor:'copilot'}), NOT request.model. Enumerate the
available families and pick a main model and a cheaper one by preference order — do
not hardcode model names, they are not API model strings. Use model.maxInputTokens
and await model.countTokens() for budgeting, never character counts. Handle
LanguageModelError: NoPermissions means stop and do not retry; quota or rate
messages mean back off and requeue. Show a status bar item with the remaining job
count, and offer resume on activation when pending jobs exist.

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
- Render "Notes from the team" sections visually distinctly from generated content,
  so readers can tell human-verified text from model output.
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
2. Run **CodeWiki: Build wiki**, selecting that repo
3. Adjust `codewiki.exclude` if the output shows generated or vendored code

To change the tool itself later, the prompt is always "read docs/DESIGN.md, then
make this change" — a diff against existing code, never a rebuild.

### One thing worth doing to the codebase itself

Annotate the abstract base classes only — one signature per method on each base.
Roughly 20 lines of work, and every subclass inherits the clarity for readers, for
Pylance, and for every digest that mentions the contract. Highest leverage per
character available, and independent of this project.
