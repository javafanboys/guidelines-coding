C/native-interop memory-safety checklist - maintained notes, not a tutorial. Read this before
writing or reviewing any plain-C code, or any Java/TeaVM code that touches `org.teavm.interop.
Address` or another language's raw-pointer FFI layer. Not a Rust-borrow-checker clone (deliberately
out of scope - see `littlejlib/jr` PRP-11) - a short list of checkable, buildable-today habits that
catch the bug shapes actually seen in this codebase, so Claude catches them before the user has to.
Discovered/confirmed 2026-09-14 auditing `littlejlib/jr`'s `launcher.c`/`javainstall.c` and the
TeaVM port's WinAPI bindings; `littlejlib/jr/prp/11-prp.03.memory-safety.md` is the worked example.

## Every fixed-size buffer copy must be bounds-checked at the point of copying, not by an
upstream invariant you're trusting from memory
`strcpy`/`strcat`/`sprintf` into a stack buffer (`char buf[MAX_PATH]` and similar) is only as safe
as "every caller's input is also within that size" - an invariant that is easy to state, easy to
believe, and never actually enforced at the call site. Use a bounded copy (`strncpy` with an
explicit size AND an explicit null-terminator write, `snprintf`, or one project-standard helper
like `safeCopy(dest, destSize, src)` - see `jr/launcher.c`) everywhere, even where the input
"can't" be too long today. This is not about the specific overflow being exploitable right now -
it's that the unchecked call site is a landmine for the NEXT change that relaxes the upstream
bound (e.g. Windows long-path support past MAX_PATH=260). **`strncpy` is not automatically safe
just because it takes a length argument** - `jr/launcher.c` also had `strncpy(dirPath, jarPath,
dirLen)` where `dirLen` was computed from `jarPath` itself (a substring length) and never clamped
against `sizeof(dirPath)`; `strncpy` only respects whatever length YOU pass it, it does not know
the destination's real size unless that length is already clamped to it. Check every `strncpy`
call's length argument against the destination's `sizeof`, not just the presence of `strncpy` over
`strcpy`.

## A function that returns a `malloc`'d pointer puts a silent, unenforced obligation on every
caller to free it - prefer a shape that doesn't need the obligation at all
Before writing `char* getX(...)` that mallocs and returns, check whether the codebase already has
the better shape for the SAME kind of problem: `void getX(char* outBuf, size_t outBufSize, ...)`,
writing into a caller-supplied buffer. `jr/javainstall.c`'s `jiSha256File(path, outHex, outHexSize)`
is the existing correct example in that codebase - it returns nothing that needs freeing. If
ownership transfer is genuinely unavoidable, name it so the obligation is visible at every call
site (an `_owned`/`_alloc` suffix or a one-line doc comment) and grep every call site to confirm a
matching `free` exists on every exit path before considering the function done.

## Every `malloc`/`calloc`/handle-acquire needs a matching free/close on EVERY exit path, including
error paths - the single `goto cleanup:` exit point is the pattern that gets this right by
construction, prefer it over free-before-each-return
`jr/javainstall.c`'s `jiSha256File` is the model: one `cleanup:` label, everything releasable
declared NULL at the top, one block of `if (x) release(x);` lines, every early exit is `goto
cleanup;` rather than its own `return`. This makes "did I free everything on this path" a single
block to read instead of an audit of every return statement. Where a function instead frees
before each individual `return` (also seen and also correct in this codebase, e.g. the `margv`
builder in `jr/launcher.c`), the thing to actually check is that EVERY return statement in the
function has its own free line before it, not just the common-case one.

## The TeaVM/Java-side equivalent: an `Address` is only as safe as the Java reference that keeps
its backing data reachable - keep that reference LOCAL, in the same method, for as long as any
native code might touch it
Java's own memory (fields, arrays, objects) is GC-safe with no action needed - the ENTIRE unsafe
surface in a TeaVM native-interop port is the `Address` values obtained from
`Address.ofData(byte[])`/similar. The shadow-stack GC (see `../guidelines-java/teavm.md`'s
`teavm_callSites` note) keeps a LOCAL variable's backing array alive across a synchronous native
call in the same method - that's what makes the existing code in `jr`'s TeaVM port safe today, not
a general guarantee. The pattern to avoid: obtaining an `Address` and then storing it (in a field,
a static, or returning it from a method) separately from a live reference to the Java array/object
that owns its backing memory - nothing signals that mistake back to you, since `Address` carries no
type-level link back to what it points into. **Since 2026-10-08 this IS enforced at compile time by `../../jarrunner/teavm-native-check` (repo jarrunner/teavm-native-check, on Maven Central as `io.github.jarrunner:teavm-native-check`), a javac plugin any TeaVM C-backend project can use through its arguments (jr PRP-35).** It fails the build on: a pointer kept in a field or returned out of a scope, `Address.ofData`/`ofObject`, pointer arrays, raw access or pointers made from numbers outside `@Unsafe("reason")` code, one struct passed for another (`@CType` from jextract_teavm), fiber suspension inside a scope, and a borrowed pointer parameter returned, stored or kept (`@Returned`/`@Escapes`, the Rust-signature idea). Bounds are checked by `Buf` in checks builds only. How-to: jr's `jr/jr/README.md`, section "Native memory"; rules and reasons: `jarrunner/prp/35-prp.status.md`. The convention it enforces (originally written before any tooling):
an `Address` should never outlive the scope of the local variable whose backing store it points
into; if a native handle genuinely needs to survive past one method call (a session/connection
handle, for instance), keep it as the plain integer/handle value WinAPI itself hands back, not as
an `Address` into Java-owned memory.

## Every `@Import`ed native function's parameter types should be verified against the real header,
not pattern-matched from the type's name - and this one IS enforceable, not just a convention
See `littlejlib/jr/teavm-poc/wintype-poc/` (PRP-11): a real `javax.annotation.processing.Processor`
that asks the actual compiler (`__builtin_classify_type`, cross-checked against known-pointer/
known-integer references, plus `sizeof` for width) whether a WinAPI typedef is pointer-shaped or
integer-shaped, and FAILS THE BUILD if a hand-written `@Import` parameter disagrees - caught PRP-09's
actual WPARAM/LPARAM-as-`Address` bug and a synthetic DWORD_PTR-as-`int` width bug, with zero false
positives on genuinely pointer-shaped types (HWND). Reach for this pattern (or port it) before
hand-declaring a new native binding's parameter types by guessing from the typedef's name.

## What to actually DO with this file, going forward
When about to write or review C code, or Java/TeaVM code with `Address`/native-interop, run down
this list before calling the work done - the same way `../guidelines-java/teavm.md` gets read before any
new TeaVM C-backend code. This is what "Claude Code should catch a class of such errors, not the
user" means in practice: a fixed checklist applied every time, not a one-off audit.
