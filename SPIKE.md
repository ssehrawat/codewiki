# CodeWiki — Environment Spike

**Run this before Prompt 1.** It answers four yes/no questions about what your VS
Code environment permits. Each has a different consequence, and one of them would
force a redesign if you discovered it late.

Nothing is installed. It is a throwaway folder plus a temporary second VS Code
window; delete the folder when you are done.

---

## What it settles

| Command | Question | If it fails |
|---|---|---|
| `spike.python` | Can the extension spawn Python? | **Redesign.** The Python engine would have to be rewritten in JavaScript, losing pytest coverage and headless operation. |
| `spike.models` | Can your extension see Copilot models? | **Agent is dead.** Fallback: Prompt 1's index only, read via `#file:` in Copilot Chat. |
| `spike.request` | Does a completion return? | Same as above. Also confirms `countTokens`, needed for budgeting in Prompt 3. |
| `spike.tools` | Does the model call tools? | Degraded. The agent falls back to retrieval-only, which Prompt 2 already handles. |
| `@spike-test` | Can your extension register a participant, and is `request.model` populated? | Chat surface unavailable; commands only. |

**Run `spike.python` first.** It is the one that would invalidate the architecture
rather than merely reduce it.

---

## Step 1 — create the folder

Make a new, empty folder **outside** the repo you plan to index, e.g. `C:\dev\spike`
or `~/dev/spike`. In VS Code: **File → Open Folder**, select it. It must be the
workspace root, not a subfolder of something else.

---

## Step 2 — have Copilot write it

Open Copilot Chat in that folder and paste:

```
Create a minimal VS Code extension in this folder using plain CommonJS JavaScript —
no TypeScript, no npm, no compile step, no dependencies.

package.json:
  main "./extension.js", engines.vscode "^1.95.0",
  activationEvents ["onStartupFinished"],
  commands: spike.python, spike.models, spike.request, spike.tools
  a chatParticipants entry with id "spike.participant" and name "spike-test"

.vscode/launch.json: one configuration of "type": "extensionHost" named
"Run Extension", with args ["--extensionDevelopmentPath=${workspaceFolder}"].

extension.js — log everything to a vscode.window.createOutputChannel, and wrap each
command in try/catch showing err.message and err.code:

  spike.python
    require('child_process').execFile('python', ['-c', 'print("bridge ok")'], cb)
    Show the stdout, or the full error if it fails.

  spike.models
    const models = await vscode.lm.selectChatModels({ vendor: 'copilot' })
    Log the count, and for each model its family, id, vendor and maxInputTokens.

  spike.request
    Send [vscode.LanguageModelChatMessage.User('Reply with the single word: ok')]
    to the first model. Collect the streamed text from res.text and show it.
    Also log await model.countTokens('hello world') and model.maxInputTokens.

  spike.tools
    Send 'How many lines are in src/main.py? Use the tool, do not guess.' with
    one declared tool:
      { name: 'get_line_count',
        description: 'Return the number of lines in a source file.',
        inputSchema: { type: 'object',
                       properties: { path: { type: 'string' } },
                       required: ['path'] } }
    Iterate res.stream, count instances of vscode.LanguageModelToolCallPart,
    and log how many tool calls came back plus their inputs. Log any text too.

  chat participant handler
    Stream back whether request.model is populated and its family name, then send
    'Reply with the single word: ok' through request.model and stream the reply.

On activation, log vscode.version, whether vscode.lm and vscode.chat are present,
and whether GitHub.copilot and GitHub.copilot-chat are installed and active.
```

---

## Step 3 — run it

1. Open the **Run and Debug** panel (play icon in the left sidebar)
2. Select **Run Extension** from the dropdown
3. Press **F5**

A second window titled `[Extension Development Host]` opens. Everything below
happens **in that window**, not the original.

If F5 opens a terminal instead of a second window, the active file is a Python file
or `launch.json` is missing — check that the `"type"` is `extensionHost` and that
"Run Extension" is the selected configuration.

---

## Step 4 — the four commands

In the dev host window, `Ctrl+Shift+P` → run each in this order:

**1. `Spike: python`** — expect `bridge ok`.
Failure here is the significant one. If the error mentions permissions, policy, or a
blocked binary, stop and reconsider the language split before building anything.

**2. `Spike: models`** — expect a non-zero count with family names.
An empty list is **ambiguous**: models appear lazily, so send any message in Copilot
Chat first, then re-run. Only if it is still empty is extension access genuinely
restricted.

Note the family names — Prompt 3 needs them to pick a main model and a cheaper one
for the bulk digest pass.

**3. `Spike: request`** — expect a consent dialog on first run, then `ok`.
Declining raises `NoPermissions`, which is a policy answer, not a transient error.

**4. `Spike: tools`** — expect at least one tool call.
Zero tool calls means the agent needs the retrieval-only fallback. Not fatal, but
worth knowing: it is the difference between an agent that verifies claims against
live source and one that paraphrases the wiki.

**5. Chat participant** — type `@spike-test hello` in the dev host's Copilot Chat.
Expect a reply confirming `request.model` is populated with a family name.

---

## Step 5 — record and clean up

Write down:

- [ ] Python spawn works
- [ ] Model count and family names: ______________________
- [ ] `maxInputTokens` of the main model: ______________
- [ ] Completion returns
- [ ] Tool calls returned: ______
- [ ] Participant registers, `request.model` populated

Then close the dev host window and delete the folder. Nothing was installed and
nothing persists.

---

## Interpreting the result

**All five pass** — build as specified. Put the model family names into Prompt 3's
configuration.

**Python spawn blocked** — the architecture changes. The engine cannot run as a
subprocess, so the AST work would have to move into JavaScript, giving up pytest
coverage and standalone CLI use. Worth pausing on before proceeding.

**Models empty after retrying** — the chat agent is not available to your extension.
Prompt 1 still stands on its own: it needs no Copilot at all, produces the index,
class hierarchy, override matrix and node DAG, and writes plain markdown that
Copilot Chat can read directly via `#file:.codewiki/wiki/<page>.md`. You lose the
tool-driven verification and clickable citations; you keep the map.

**Tools return zero calls** — build as specified. Prompt 2 already includes the
fallback; just confirm the answer footer states which mode produced it.

**Participant fails but commands work** — expose the agent through a command that
opens a results document instead of the chat panel. Same engine, different surface.
