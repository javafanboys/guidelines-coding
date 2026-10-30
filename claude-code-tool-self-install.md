# How a CLI tool installs itself into Claude Code

Written 2026-10-30 from experiments with jacli (github.com/littlejlib/jacli), which is the worked example: its `jacli claude print|install|uninstall|status` verbs implement everything below. Other CLI tools should copy the contract, not the code line by line.

## Why a tool installs itself

A tool that relies on the user pasting a paragraph into their CLAUDE.md is invisible on every machine where nobody did. The tool already ships to that machine; the prompt and hooks can ship inside it. No MCP server: The jacli experiments measured an MCP server with all five tools provably presented and ignored 0 of 3 times, and it costs tokens on every turn.

What the jacli experiments measured, in one line each, decides what to embed:

- Only text the model sees BEFORE it starts changes its choice: CLAUDE.md, an appended system prompt, a UserPromptSubmit hook.
- A skill was never invoked in a clean sandbox (0 Skill calls in 3 runs). An MCP server was ignored. Repeating a hint after every tool call (PostToolUse, 22 times per run) did nothing.
- A PreToolUse hook only fires when the model issues the command it watches. It is the right tool for stopping a known bad reflex (jacli: `find /` for a jar, `unzip` + `javap`), not for introducing the tool.

## What the tool embeds (as resources in its own jar or binary)

1. `prompt.md`: the section a user would otherwise paste into their global prompt. Short, because at user scope it is in every session. Start it with an HTML comment saying which command wrote it and which removes it.
2. `hooks.json`: the settings.json `hooks` fragment, exactly as it will be merged.
3. The hook texts (one file per message) and the hook logic, inside the tool, reached through a hidden verb (`<tool> claude hook <event>`).

## The verbs, and the contract each must keep

- `<tool> claude print [--prompt|--hooks]`: prints what install would write, for a user who wants to paste it by hand or read it first.
- `<tool> claude install [--scope user|project] [--config-dir <dir>] [--project-dir <dir>] [--dry-run]`
  - Writes the prompt as its OWN file and adds ONE import line to CLAUDE.md. Never edits the user's prose.
  - Merges hooks into settings.json leaving every other key and hook as found, in the same order, with the same number text.
  - Idempotent: a second run writes nothing at all (compare the bytes before writing; skip when equal). A stale install is replaced, not duplicated.
  - Prints each file it wrote, and says a running session will not pick it up.
- `<tool> claude uninstall`: removes exactly what install added. A CLAUDE.md or settings.json left empty is deleted (install may have created it); at project scope an emptied `.claude/` folder is deleted too. Proven byte for byte by a test that installs over existing content, installs again, uninstalls, and compares.
- `<tool> claude status`: one line per piece (prompt, import, hooks) as absent / current / stale, and a note listing any OTHER hooks that mention the tool but are not managed by install (hand-written ones from before). Exit 0 current, 1 stale or partial, 3 absent, so a script can act on it.
- Recognise your own hook entries by a marker string in the command (`jacli claude hook`), not by a separate manifest that can fall out of step with the file.

## Where each piece goes

User scope (every session of that Claude Code configuration):

- config dir = `--config-dir`, else `CLAUDE_CONFIG_DIR`, else `~/.claude`
- prompt: `<config>/<tool>.md`; import line in `<config>/CLAUDE.md`: `@<tool>.md` (a relative import resolves against the importing file: verified in the sandbox)
- hooks: `<config>/settings.json`

Project scope (one project, committed with it):

- prompt: `<project>/.claude/<tool>.md`; import line in `<project>/CLAUDE.md`: `@.claude/<tool>.md`
- hooks: `<project>/.claude/settings.json`

## How the hooks are written

- The command in settings.json is a short bash prefilter. Claude Code runs hooks through bash on every platform, Git Bash on Windows, so bash is the one dependency that is always there. jq is NOT: it is absent on a stock Windows or macOS machine, so the hook logic must not need it.
- The prefilter starts the tool only when a trigger word is in the hook input: `in=$(cat); case "$in" in *javap*|*unzip*) printf '%s' "$in" | jacli claude hook pre-bash ;; esac; true`. A JVM tool costs ~600 ms per start, which is fine once per prompt and not fine on every Bash call.
- The command uses the tool's name from PATH, never an absolute path of the machine it was installed on. `status` should still be read with that in mind: a hook calling a tool that later leaves PATH fails silently.
- The hook verb never fails loudly: any error is swallowed into exit 0 with nothing printed. A broken hook should cost a missing hint, never a blocked tool call.
- End with `; true` so the prefilter's own exit status is never read as a hook failure.

## Traps found while building it

1. **The ancestor walk leaks the user's global prompt into a "clean" sandbox.** `CLAUDE_CONFIG_DIR` isolates the config dir, but Claude Code still reads `CLAUDE.md` and `.claude/CLAUDE.md` in every ancestor of the working directory. A sandbox project under the user's home folder therefore loads `~/.claude/CLAUDE.md` as if it were a project file. An earlier experiment ran its "clean" control in a folder under the home directory, so it carried one line of the global prompt (the import line, which was not expanded). Put sandbox projects outside the home tree (for example `C:\sandbox\runs\...`).
2. **Folder names are read by the model.** The working directory and the memory path are in the system prompt. A sandbox folder named after the tool tells the bare arm the tool's name. Use neutral names.
3. **A child `claude` started from inside a Claude Code session inherits that session's identity** (`CLAUDECODE`, `CLAUDE_CODE_SESSION_ID`, the messaging socket and token). Strip them with `env -u ...`.
4. **A fresh `CLAUDE_CONFIG_DIR` is not logged in.** Copy one account's `.credentials.json` into it for the experiment and delete it after. The snapshot does not rotate with the live login, so pick the account with quota to spare.
5. **From Git Bash, write `CLAUDE_CONFIG_DIR` with forward slashes** (`C:/sandbox/inst`). The backslash form was silently not used: the session reported "Not logged in" and wrote nothing into the folder.
6. **Prove the sandbox with a positive control, not only a negative one.** A probe that asks the model to quote lines containing a marker returns NONE both when the sandbox is clean and when the probe is broken. Plant a marker in the sandbox's own CLAUDE.md and check it is seen; then check the user's real markers are not. Self-report is enough for this only because the answer must quote text the question does not contain.
7. **Test inputs that look like the bad reflex trip the user's own live hooks.** Testing the `find /` deny hook from a shell command was itself denied by the machine's installed hook. Feed hook inputs from a file.
8. **Rewriting settings.json with a JSON library reformats it unless told not to.** Gson escapes `=`, `<`, `>`, `&`, `'` as `\u003d` and so on by default; turn HTML escaping off, keep the parser's insertion order, and write only when the bytes differ.
9. **The first call after an upgrade of a jr/AOT-launched tool assembles its AOT cache and takes seconds.** Give hooks a timeout that covers it (15 s here). The cache chatter goes to stderr, so the hook's JSON on stdout stays valid.
10. **A tool that answers through a daemon must run its `claude` verbs in-process**: the daemon has neither the hook's stdin nor the caller's working directory and environment.

## How to measure it for another tool

A harness of five small scripts is enough: make two config homes outside the home tree (`bare`, and `inst` after `<tool> claude install --config-dir`); probe what a session in each sees; run a task N times per arm with the parent environment stripped; count tool use from the transcript whose id the result recorded; grade against ground truth. Swap the fixture and the task; keep the arms differing only by the config home.

## Measured: jacli, bare sandbox vs after `jacli claude install` (2026-10-30)

Haiku, n=3 per arm, `--disallowedTools Agent`, two Java fixtures and their tasks. The arms differ only by config home. Bare probed clean (no CLAUDE.md, no hooks, no jacli anywhere in its context); inst probed as seeing the prompt via `@jacli.md`. "Java files read" counts the Read tool plus files named in `cat` commands, since one bare run read 13 files through `cat`.

    base5, orientation (67 files)   used jacli   jacli calls   tool calls   java files read   input tokens   cost     correctness
    bare                               0/3           0.0          13.7           11.7             188k        $0.0103     7/7 x3
    installed                          3/3           4.0           6.7            3.0             261k        $0.0110     7/7 x3

    base4, stack trace (34 files)   used jacli   jacli calls   tool calls   java files read   input tokens   cost     sibling-guard / dead-method found
    bare                               0/3           0.0           8.0            5.7             173k        $0.0114     2/3 / 2/3
    installed                          3/3           4.7           5.0            4.0             231k        $0.0105     3/3 / 1/3

The UserPromptSubmit hook fired in 6/6 installed runs and 0/6 bare.

What it says:

- Install gets the tool reached for every time, and this time it SUBSTITUTES: on the open question, tool calls halve (13.7 to 6.7) and files read fall by three quarters (11.7 to 3.0); two of three runs read no source file whole at all. An earlier CLAUDE.md-only arm reached for jacli but kept reading nine files. The difference here is the combined delivery (prompt plus hook, both saying "instead of"), not either mechanism alone; this experiment does not separate them.
- Input tokens rise by about a third (the prompt, the hook text and jacli's answers are all context), and cost is flat within noise. Fewer round trips did not buy fewer tokens on haiku at this size.
- Correctness: unchanged on base5. On base4 the installed arm caught the sibling-guard asymmetry 3/3 against 2/3, and named the never-called `wireTesting` 1/3 against 2/3. n=3 cannot tell either apart from noise.
- Limits: n=3, haiku only, two fixtures. Not tested on sonnet or opus; the `-T` dependency path and the deny hook were not exercised by these tasks.
