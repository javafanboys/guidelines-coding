# Secrets, private data, and git — the rules before anything reaches a remote

Scope: coding work. Read this before the first push of any repository, before making a repository public, and before any history rewrite. Written after a real leak: two repositories of a private organisation sat public for three weeks carrying a named colleague's work email, real internal paths, a live Google Sheets URL, and an internal campaign name used as an example string.

## 1. The rule

**A repository is public from the moment it is created unless you have proved otherwise, and everything ever committed to it is public — not just what `HEAD` shows.** Audit before the first push, not before the first release.

## 2. What actually leaked, and what to look for

Credentials are the obvious class and they were NOT the problem. Every `client_secret` hit was a JSON field name in OAuth-parsing code. What leaked was **operational** data, sitting in example strings, javadoc, help text and test fixtures, where nobody thinks to look:

- a real person's work email, used as sample data in a UI preview harness
- real absolute paths naming internal projects and internal output files
- a real Google Sheets id, including a full `docs.google.com/spreadsheets/d/...` URL in a README
- internal repo names referenced from documentation
- an internal campaign name used as the running example for an environment variable

None of it was load-bearing. Every one could have said `someone@example.org`, `C:\work\projects\...`, `1AbCdEf...`, `"invoice reconciliation, March"` and the code would be identical.

**"It is already public somewhere else" is never a reason to publish it again.** On 2026-10-01 `jarrunner/aot` was split out of `jarrunner/jr` and pushed with the user's own app names as its worked examples (a private app, its launcher and two of the user's tools, with their timings). The scan found them and Claude let them through because the same text already sat in jr's public history. The user caught it; the repo was deleted and recreated. A new repository is a new, separately indexed copy that turns a buried line into a headline. Judge each name on its own: an internal app or tool name is private data even when the code is open. Use `com.example.myapp`, "a CLI tool", "a desktop app", and keep the measurements. Even a tool that is itself public (`proj`) is better replaced by a vague example or left out.

**So scan for both classes.** Credentials: `ghp_`, `github_pat_`, `AIza`, `GOCSPX-`, `BEGIN [A-Z ]*PRIVATE KEY`, `refresh_token`, `rediss://`, `.env`, `*.p12`, `*.pem`, `*.key`, `credentials.json`. Private data: every email that is not obviously a placeholder, every id of 25+ characters, every absolute path containing a username or a project name, every internal repo or tool name, and every organisation-specific term that would read badly quoted back.

**Scan HISTORY, not HEAD:** `git grep -I -l -E "<pattern>" $(git rev-list --all)`.

**When listing candidates, never truncate a sorted list.** A `sort -u | head -12` over "long strings" showed twelve rows of dashes and hid the one real spreadsheet id underneath. That single `head` is why a second repository was declared clean when it was not.

## 3. Why this matters more than it looks

For work touching legal, media, or advocacy, the damage is not a compromised credential — it is a **quotable sentence**. A public repo stating there is a batch process directed at a named news organisation, with a named person's email attached, is a headline someone else writes for you. Judge every example string by how it reads out of context, in a hostile summary.

## 4. `.gitignore` must carry these before the first commit

`prp` · `.claude` · `CLAUDE.md` · `target/` · `shade/` · `dist/` · `credentials.json` · `*.key` · `*.pem` · `*.p12` · `*.log` · `*.jar`

`prp` matters most: conversation logs contain everything — real ids, emails, paths, internal reasoning. Verify with `git log --all --pretty=format: --name-only | grep '^prp/'` returning nothing, rather than trusting that the ignore line was there from the start. **Note `CLAUDE.md` is commented out in some existing `.gitignore` files in these projects.** Uncomment it or confirm the file does not exist.

## 5. A force-push does NOT delete anything

This is the part that is routinely got wrong, so measure it rather than assume:

- Rewriting history and force-pushing moves a branch pointer. The old commits become unreferenced; they are **not deleted**. GitHub still serves them by direct SHA through the web UI and the API. This was verified: after the rewrite, `GET repos/OWNER/REPO/contents/<path>?ref=<old-sha>` still returned the confidential line.
- The old SHAs are **public**. Every `PushEvent` on a public repository carries `payload.head`, and that event stream is archived permanently by third parties. Nobody needs to have cloned the repo to obtain a SHA.
- GitHub's clone/view traffic API retains only **14 days**, so "zero clones" says nothing about an exposure window longer than a fortnight.

**Therefore: a rewrite makes a repo browse clean; only deleting and recreating the repository (a new object store), or a GitHub Support purge request, actually removes the data.** Check `forks`, `stargazers`, `subscribers`, `issues`, `pulls`, `releases` first — if all zero, delete-and-recreate costs nothing, keeps the URL, and is decisive.

## 6. Before rewriting: back up, and verify from the remote afterwards

`git bundle create <name>.bundle --all` plus a copy of every modified and untracked file. A bundle restores the pre-rewrite state exactly.

Verify by **cloning fresh from the remote** and scanning that. Local verification proves only what your machine believes.

## 7. Three traps when scripting a scrub — all three were hit

1. **An apostrophe inside a shell single-quoted `perl -e` program closes the string early**, silently dropping every rule after it. A comment reading `# any other 'ACR batch:' variant` cost two-thirds of the rules, and the dry run still looked plausible because the surviving rules did fire. Keep apostrophes out entirely.
2. **A negated character class eats the escaping backslash.** A Java literal ends `\"`, so `s/PREFIX:[^"]*/.../` consumes the backslash and produces an unclosed string literal — invisible until compile. Use an explicit safe class (`[A-Za-z0-9 :,-]*`).
3. **Backslash levels do not survive a heredoc reliably.** Write patterns and replacements that contain **no backslashes at all** — replace tokens (an internal folder name, an internal org name) rather than whole paths, and the separators stay untouched.

And always: dry-run on copies, sweep with word boundaries (`acr` matches "across"), then **compile** before pushing.

## 8. Tokens

Never in a `.bat`, a script, or a command line — they land in shell history and in claude-code transcripts permanently. Use an environment variable or a credential wallet. If one has been in a file, treat it as compromised and rotate it.

## 9. The routine check, every time, before `git push`

```
git log --all --pretty=format: --name-only | sort -u | grep -iE "credential|secret|token|\.env|\.pem|\.p12|^prp/|CLAUDE\.md"
git grep -I -l -E "ghp_|github_pat_|AIza|GOCSPX-|BEGIN [A-Z ]*PRIVATE KEY|refresh_token" $(git rev-list --all)
git grep -I -h -oE "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}" $(git rev-list --all) | sort -u
git grep -I -h -oE "\b1[A-Za-z0-9_-]{25,}\b" $(git rev-list --all) | sort -u
```

Read every line of the last two. Placeholders only.

