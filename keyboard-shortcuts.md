# Keyboard shortcuts — orthogonality

Read this before binding any key in any UI: JavaFX, Swing, a TUI, a web app, a CLI prompt. It is short on purpose.

## The principle: a key means one thing, everywhere in that window

The measure of a keyboard UI is not how many shortcuts it has. It is whether the user can reach any action **without first working out what mode they are in**. That holds only if each key belongs to exactly one job, and no key has to be shared with typing.

So partition the keyboard into families, and let each family own a job nothing else touches:

- letters, digits, space, backspace — **typing**. They go to the filter, the search box, the name field. Nothing else may claim them.
- arrows, PageUp/Down, Home/End — **moving the selection**.
- function keys — **actions**. F1..F12 are unreachable by typing, so they work from wherever the focus is, with no modifier.
- Enter — commit. Esc — cancel. Never anything else.
- Alt+letter — the leftovers, for secondary toggles only, once the function keys are genuinely used up.

**The test, and it takes one second: could a user type this key while entering a name?** If yes, it cannot be a shortcut in that window. "Auxiliary" cannot own A when there is a search box on screen, because a project called `atlas` exists.

A window with **no text input at all** — a pure menu of buttons — is the one exception: bare letters are fine there, because nothing is competing for them. Say so in a comment, so the next reader does not "fix" it into inconsistency.

## Why MS-DOS got this right

Turbo Pascal, Norton Commander, WordPerfect, edit.com — all driven at speed with no mouse, by people who never read a manual. They worked because the key families were stable and visible: F-keys did the actions, the legend along the bottom of the screen named them, and the legend was always on screen rather than hidden in a menu. Nothing needed a chord for the common path.

Two habits worth copying:

- **A permanent shortcut legend** in the status bar. A key you have to look up somewhere else is a key read the long way round every time.
- **Wear the key on the control.** Label the button `F4 Path`, not `Path` — then the legend is a reminder rather than the only source.

## The legend is a control, not a caption

**Every shortcut a window offers must be visible in that window, and every item in that legend must be clickable.** Both halves are requirements, not refinements.

Visible, because a key the user cannot see is a key they do not have. A shortcut discovered once and then forgotten is worse than no shortcut: it was paid for in implementation and returns nothing. The legend goes where the eye already is - along the bottom of the pane, or on the strip the keys act upon - and it stays there rather than living behind a help dialog. Where several panes bind the SAME key to different actions, show the focused pane's legend at full strength and fade the others, so the legend also answers "which pane owns this key right now".

Clickable, because the legend is already a complete, always-correct list of what this window can do - which is the definition of a toolbar. Making it inert means building that list twice: once as text the user reads and once as buttons they press. It also strands anyone whose hands hurt today and who wants the mouse for a moment, and anyone who has just discovered the action by reading the legend and whose next instinct is to click the thing they are looking at. A legend item and its key must invoke ONE function, never two implementations of the same action.

The two together are also the cheapest guard against the rule below: a legend built by iterating the same enum the key handler switches on cannot advertise a key nobody bound, because the entry and the binding come from one declaration. Hand-written legend strings can and do drift; generated ones cannot.

## Never advertise a key you did not bind

A button labelled `Copy (P)ath` whose P is wired to nothing is worse than an unlabelled button: it teaches a key that silently fails. When shortcuts get reworked, re-grep the labels. Same for tooltips.

## The hands constraint (this user specifically)

Combination keys and mouse use are painful for some of us (a hand/nerve condition). Single keys are much easier. So: the common path must be single keys, always. Alt/Ctrl combos are acceptable only for infrequent actions (a scope toggle set once a session, a Save in a rarely-opened settings dialog), never for anything on the main loop.

## The JavaFX trap — mnemonics stop asking for Alt

`new CheckBox("_Auxiliary")` looks like the classic DOS Alt+A mnemonic. It is not. In `KeyboardShortcutsHandler` the condition is:

```java
if (event.isAltDown() || isMnemonicsDisplayEnabled()) processMnemonics(event);
```

`mnemonicsDisplayEnabled` is a **latched flag on the Scene** — the first Alt press in that window turns it on and nothing reliably clears it. From then on the **bare** letter fires the mnemonic, consumes the event, ticks the box and moves the focus, and the letter never reaches the filter box. It presents as an intermittent bug nobody can characterise: fine on a fresh window, broken after an Alt-Tab. A resident daemon that reuses one Scene latches it once and keeps it latched for days.

Fixes, in order:

1. Do not put `_` in any control label. `Labeled.mnemonicParsing` defaults to **true** for Button, CheckBox, RadioButton, ToggleButton, Hyperlink and MenuItem (false only for Label), so the underscore alone is enough to register the mnemonic.
2. Sweep the dialog root at build time and set `setMnemonicParsing(false)` on every `Labeled`, recursing into `TabPane` tab contents by hand — they are built lazily and a plain child walk misses them. `littlejlib.proj.ui.Keys` is a working ten-line implementation.
3. Bind Alt yourself in a scene-level `addEventFilter(KEY_PRESSED, …)`, checking `isAltDown() && !isControlDown() && !isShiftDown()`. A scene filter runs before the controls see the key, so it also keeps a table's built-in type-ahead out of the way.

Swing's `setMnemonic` is honest by comparison (always requires Alt), but the design rule above stands regardless of toolkit.
