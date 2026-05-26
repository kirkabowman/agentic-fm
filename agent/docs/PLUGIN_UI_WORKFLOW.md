# Plugin UI Workflow — Announce and Go

When working with the AgenticFM plugin's UI-control endpoints (script mutation, AX clicks, focus-dependent bridge sequences), the agent **must announce the operation and wait for the developer to confirm before running it**.

## The pattern

Before any plugin call that takes UI control of FileMaker:

> **Stay off keyboard.** Say "go" to [redeploy / open / paste / press / whatever].

The developer types `go` when their hands are clear of the keyboard. The bash command then runs unimpeded by stray keystrokes.

## When it applies

Any plugin operation that drives FileMaker's UI:

- `POST /api/ui/script/insert` — insert steps
- `POST /api/ui/script/delete` — delete steps
- `POST /api/ui/script/select` — select steps (when followed by delete)
- `POST /api/ui/script/create` — create a new script
- `POST /api/ui/script/save` (after destructive ops)
- `POST /api/ui/script/navigate` (less critical but still UI-driven)
- `POST /api/ui/press` — AX click on UI elements
- `POST /api/ui/set` — set values on UI controls
- `POST /api/ui/inspector` — Inspector panel manipulation
- AGFM_Bridge dialog commands (`scriptWorkspace`, `manageDatabase`, etc.) when followed by another UI action
- AGFM_Bridge UI sequences like `selectWindow` + `saveAsXml` where focus matters
- Legacy Tier 2 deploy flows (largely replaced by plugin endpoints)

## When it does NOT apply

Read-only operations that don't touch UI focus:

- `GET /api/ui/script` — read current script
- `GET /api/ui/environment` — window/state snapshot
- `POST /api/discovery/*` — indexed-data queries
- `POST /api/eval`, `POST /api/query` — calc / SQL evaluation
- `GET /api/clipboard` — read clipboard
- `POST /api/performscript` for non-UI scripts

When in doubt, announce. The cost of waiting 2 seconds is negligible; the cost of recovering from a focus-drift bug is much higher.

## Why this exists

Stray keystrokes during plugin UI automation cause documented failure modes:

1. **Script doubling** — `select` + `delete` partially applies, `insert` then appends instead of replaces. The script ends up with the old body AND the new body concatenated.
2. **Focus drift** — `paste` lands in the wrong field, wrong window, or wrong script tab.
3. **AX automation failures** — the button under the cursor moves before the click registers, the click misses.

All three were observed during multi-phase script-deploy work. Adopting announce-and-go eliminated them.

## Convention

The announcement should be a single short sentence — no preamble, no explanation. Examples:

- `**Stay off keyboard.** Say "go" to redeploy.`
- `**Stay off keyboard.** Say "go" to paste the snippet.`
- `**Stay off keyboard.** Say "go" — about to clear and rewrite the script.`

The developer's `go` is the only signal needed. Anything else (including `ok`, `proceed`, `yes`) is also fine; treat any clear affirmative as permission.

## Front-load processing; minimize UI-control duration

When the next operation is going to take UI control of FileMaker, do **all** the prep work **before** the "Stay off keyboard" announcement:

- HR → XML conversion (`/api/hr-to-xml`)
- Validation (`/api/validate`)
- XML patching (e.g. fixing `Restore=False` on Perform Find, swapping `<DialogMessage>` → `<Message>`)
- Step-count expectations (so the verification step is a simple equality check)
- File reads, decisions, lookups — anything that can be computed without touching FM

The bash that runs after `go` should be a tight sequence:
1. read current state
2. select all
3. delete
4. insert
5. save
6. verify

Nothing else. No conversion, no lookups, no thinking. The developer's hands are off the keyboard; that window is precious.

## Tighten delays where feasible

Polling and sleeps between UI ops should be tuned, not guessed. Some observed patterns:

| Operation | Typical safe delay | Notes |
|---|---|---|
| `selectWindow` → next op | 3–4s | Binding settle; could be polled via `/api/ui/environment` until frontmost matches |
| Open Script Workspace → next op | 2s | Could poll `summary.scriptWorkspaceOpen` until true |
| `delete` → `insert` | 1s | Could poll `stepCount` until 0 |
| `insert` → read | 1s | Read may return stale state; small wait helps consistency |
| `saveAsXml` | dispatch + poll eval id | Already polled — no fixed sleep |

When adding new sequences, **prefer poll-until-condition over fixed sleeps**. Use fixed sleeps only when no observable condition is available, and revisit them periodically to tighten.

## Origin

Adopted 2026-05-25 after observing repeated script-doubling errors during Phase 3 / Phase 4 deploys. The developer requested codifying the workflow:

> "For actions where the plugin needs to control the FileMaker UI, do all the thinking/processing up front so the UI control takes minimal time. Also, if there are any delays to allow time for the UI to perform an action, monitor and tighten those delays as feasible."
