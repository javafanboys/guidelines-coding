# General coding guidelines — the master file

Read before any coding work. Java has its own master file, `../guidelines-java/java.md`. Paths here are relative to this file; repositories sit at `<codedir>/<org>/<repo>`, mirroring GitHub.

- # Language and format preferences
	- DEFER PYTHON : generally i do not want to use python as it is new to me. **NEVER USE PYTHON FOR CODING** - WE DO NOT EVEN HAVE PYTHON INSTALLED.
	- OTHER choices for general coding - javascript ok, c/c++ is fine, vba also ok.
	- JAVASCRIPT / TYPESCRIPT : prefer using bun over npm, pnpm etc. PLEASE NOTE : `bun.exe` is installed and in path, you can use it. 
	- **CONFIGURATION FILE FORMAT** : Across all programming languages and projects, the standard format is as follows :
		- prefer `.json` wherever the data is read/written by machine (serialized state, API payloads, inter-process data); i think in most cases `.json` will win, there is no point of writing first in `.toml` and if data gets complex migrate to `.json`, rather json can be a sensible default. 
		- prefer `.toml` when the config represents intent or settings a human might read or tweak (app config, paths, feature flags). 
		- `.xml` is acceptable. Situations where XML genuinely wins over json is when comments are a first-class requirement (e.g. commenting out entries, inline documentation within the config), or when the data has a natural document/element-with-attributes structure that would be awkward in JSON. Also note we will continue to have pom.xml etc.
		- `.properties` is not preferred — backslash escaping for Windows paths is error-prone, and Unicode requires \uXXXX escapes, additionally it doesn't have nesting or structure. Exceptions: JVM-mandated files like logging.properties or jbang zero-dep scripts with 2–3 flat keys.
		- `.yaml` is not a project choice — only when an external tool mandates it (e.g. GitHub Actions, Docker Compose).
		- plain text and markdown is fine sometimes, especially for prompts and other text specific data. 
		- html is fine for templates 
		- tsv is fine for tabular data, more preferred than csv 
		- this list is indicative, not exhaustive, we are not listing every format under the sun here, because that amount of detail is not needed at this level. 

- # Read when needed
	- **MANDATORY RULE — WEB UI** - DO NOT REPORT A WEB PAGE AS DONE WITHOUT READING AND FOLLOWING `webui.md` (in this folder) ; applies to ANY html/css a human will look at (static site generator, server-rendered page, JS app, html email body), in any language. A clean compile, a typed markup DSL and a green test suite all prove only that the markup is what you wrote — none of them can see that what you wrote is wrong. The file carries the two checks (walk the links, screenshot and READ it), the chrome-headless traps, and the catalogue of defects that reach production having compiled clean.
	- **MEMORY SAFETY** - before writing or reviewing any plain-C code, or any Java/TeaVM code touching `org.teavm.interop.Address` or another raw-pointer FFI layer, READ `c-memory-safety.md` (in this folder) . A short, checkable list (bounded copies, caller-supplies-the-buffer over ownership-transfer mallocs, single-exit-point cleanup, Address-locality on the TeaVM side, header-verified `@Import` types) grounded in real bugs found in `littlejlib/jr` (PRP-09 through PRP-11) - not a Rust-borrow-checker clone, deliberately.
	- **KEYBOARD SHORTCUTS** - before binding ANY key in any UI (javafx, swing, TUI, web, cli prompt), READ `keyboard-shortcuts.md` (in this folder) . One rule, one test: a key that can be typed into a search/name box may never also be a shortcut. Letters and digits belong to typing; F-keys do the actions with no modifier; Alt+letter only for infrequent leftovers; combos never on the common path (see the hands constraint). Carries the MS-DOS-era conventions worth copying, and the JavaFX mnemonic trap - a `_` in a label silently stops requiring Alt after the first Alt press in that window, which is how `"_Auxiliary"` came to eat the letter A in `proj o`.
	- **GIT: SECRETS AND PRIVATE DATA** — before any push, before making a repository public, and before any history rewrite, READ `git-secrets.md` in this folder.
	- **MAKING A CLI TOOL INSTALL ITSELF INTO CLAUDE CODE** (its agent prompt and hooks shipped inside the tool, `<tool> claude print|install|uninstall|status`) — READ `claude-code-tool-self-install.md` in this folder. jacli is the worked example.
