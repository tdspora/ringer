---
name: ringer
description: >-
  Orchestrator playbook and routing rules for Ringer, the verified-swarm
  delegation tool (ringer.py). TRIGGER — load BEFORE acting, not after —
  whenever: you are about to run ANY script or command that calls a model or
  drives a conversational/eval harness (probe, smoke test, simulation,
  grader, persona conversation) outside a live Ringer run; you are about to
  start an edit→test→edit loop or a batch of similar edits across files; you
  are about to do a "quick check" that spawns a model or a CLI agent; you are
  reviewing or diagnosing failed worker or model output; you catch yourself
  thinking a task is "small enough to just do myself" — that thought IS the
  trigger (a single task is a one-task manifest, and a bounded read-only
  question is `ringer.py ask`); or you are writing or
  reviewing a manifest, choosing a swarm pattern (review swarm, fix swarm,
  focus group, bakeoff, research-with-proof), picking a worker engine, or
  debugging a failed run. SKIP only for: reading or searching files, git
  operations, a one-file few-line ONE-SHOT edit (once — if you are back for a
  second pass, that is a loop: TRIGGER), authoring prose/specs/docs straight
  from your own context, or pure conversation.
---

# Ringer orchestrator playbook

## Read this first — the four rules that actually get broken

1. **You review; workers type.** Your lane: specs, checks, pattern choice,
   reading results. If you are typing implementation, running probes, or
   babysitting a retry loop yourself, you have left your lane.
2. **A single task is a one-task manifest.** Same verification, zero
   ceremony. "Too small for Ringer" is how drift starts — the smoke test,
   the probe script, the three-edit fix are all one-task manifests.
3. **Beware the tiny-edit death spiral.** The named anti-pattern: each step
   is individually small enough to justify inline, and two hours later the
   exception has become the workflow and nothing was verified or visible.
   The one-shot exception is ONE file, a few lines, ONCE. The second pass on
   the same problem is a loop, and loops are manifests.
4. **Runs are watched, not hidden — and the screen comes up FIRST.** The
   moment this skill loads for real work, before you write a single spec,
   put Ringside on the human's screen: `$RINGER_CHECKOUT/ringer hud` (idempotent — if
   one is already up it says so and opens the page; runs also auto-start
   it). Ringside is the PAGE at http://127.0.0.1:8700 — NEVER launch the
   Ringside.app application (`open -a Ringside`); it is a parked prototype
   with a stale frontend. And never go dark: if your prep (research,
   check-writing, manifest drafting) will take more than ~30 seconds,
   tell the human in one sentence what you're doing and roughly how long
   before you start — they should be watching the empty arena and reading
   your one-liner, not wondering if anything is happening. Never pass
   `--no-dashboard` except in automated tests or when the user explicitly
   asks.

## Where Ringer lives

Ringer is a checkout of its own, not part of the repository you are working
in. `RINGER_CHECKOUT` holds its absolute path; it is set in the `env` block of
`~/.claude/settings.json`. Never use `RINGER_HOME` for this: Ringer reads
`RINGER_HOME` as its state directory (`~/.ringer` by default), so pointing it
at the checkout puts run state and hook state inside the repository. In the
EPMC-TDM workspace the checkout sits next to the other repositories, as
`<workspace>/ringer`. Every `$RINGER_CHECKOUT/...` path in this skill is
inside it.

Call Ringer through its launcher, `$RINGER_CHECKOUT/ringer`, from the
directory you are working in (`$RINGER_CHECKOUT/ringer lint manifest.json`).
The launcher runs `ringer.py` with the Python 3.12 the checkout pins in
`.python-version`; pyenv applies that pin only inside the checkout, so calling
`ringer.py` directly from a repository gets the global Python and exits.
Staying in your own directory keeps relative manifest paths and a repository's
`.fleet-agent` identity working. If `RINGER_CHECKOUT` is empty, or
`$RINGER_CHECKOUT/ringer` does not exist, stop and work through "Setting
Ringer up" at the end of this skill before anything else.

Ringer runs manifest tasks in parallel across cheap CLI workers (OpenCode
over DIAL, others via config) and verifies every task by **executing a
check command** — exit 0 is the only PASS. Failed tasks are retried once
with the check's actual failure output injected into the retry prompt. You —
the orchestrating model — pay tokens only for specs, orchestration, and
review.

```bash
$RINGER_CHECKOUT/ringer lint manifest.json            # always lint before running
$RINGER_CHECKOUT/ringer run manifest.json --identity <who-you-are>
$RINGER_CHECKOUT/ringer run $RINGER_CHECKOUT/lane-test-dial.json --identity <who-you-are>   # DIAL smoke test
$RINGER_CHECKOUT/ringer run manifest.json --dry-run   # print the plan, spawn nothing
```

Runs land in `~/.ringer/runs/`. Raw worker logs land in `<workdir>/logs/`.
Full reference: `$RINGER_CHECKOUT/README.md`. Ready-made manifest skeletons: `$RINGER_CHECKOUT/templates/`.
Lint catches unverifiable checks, silent checks, worktree deliverable/commit
loss, serial fan-out, write collisions, and underspecified specs; `run`
prints the same findings as non-blocking warnings.

## The one exception: `ask`

Rule 2 holds for anything that changes a file, runs a build, or produces an
artifact worth checking. One lane doesn't fit it: the human asks a bounded,
read-only question over source you can already point at, and the answer is
prose. A manifest for that is ceremony — but answering it in your own context
means pulling whole files into a conversation that is already expensive.

```bash
$RINGER_CHECKOUT/ringer ask "<the human's request>" --source /absolute/path/to/source
```

`ask` selects the passages that match the request, caps the packet, spawns one
clean worker on it, and allows a single attempt. Repeat `--source` for several
files or directories; `--state` takes a small file of settled decisions;
`--dry-run` shows you the packet and spends nothing. If everything that matched
is too large for the packet it says so and stops before the model call rather
than letting a worker guess — but a source small enough to fit whole is sent
whole, relevant or not, so choosing the sources IS the work. Directory scans
stay inside the tree you name; a symlink leading out of it is skipped and
reported. Runs appear on Ringside like any other, and `--redact` hides the
request from Ringer's own state and eval records — it cannot scrub raw worker
output, which is captured verbatim by design.

**Be honest about what it verifies.** The check is that `answer.md` exists and
is non-empty. That is the weakest check in the tool, and it is also the best
available — there is nothing to execute against free-form prose. `ask` proves
the worker answered, never that the answer is right. You still read it.

**Everything else is a manifest.** Code changes, external actions, research
you intend to act on, anything whose output a check could actually execute —
those keep the full path. When a request sits near the line, the tiebreaker is
whether you could write a check that would catch a wrong answer. If you can,
write it, and make it a manifest.

## One job, one artifact

A job the human asked for — however many rounds it takes — is ONE artifact.
Use the SAME `run_name` for every round (`sd-crate-launch`, not
`sd-crate-r1` / `sd-crate-r2`): the library accumulates each round as a
version under one entry, and the human watches one page evolve instead of
hunting across three "live" tabs. Name it after the JOB in the human's
words, not after your batch structure.

And the artifact page is where results are REVIEWED. When a round finishes,
read the deliverables from the artifact store and direct the human to the
page — never `cat` result files into the terminal as the reveal. If a result
matters, it belongs in the artifact; if it isn't there, that's a harvest gap
to fix (declare it in `expect_files`), not a reason to bypass the page.

## Spec-writing craft

Workers are stateless and cannot ask questions. Every spec must be
self-contained:

- **Open with the role and the boundary.** "You are a read-only scout…",
  "Your current working directory IS a git worktree of <repo> — edit files
  here directly." State what the worker must NEVER touch before what it
  should do.
- **Name every file the worker owns.** In multi-worker runs over one repo,
  file ownership must be disjoint — and disjoint across *all* concurrent
  lanes/branches, not just within one batch. Every file a spec mentions must
  be in that worker's ownership list.
- **Embed the HOW TO RUN.** If the task drives a harness or script, put the
  exact command lines (with real absolute paths) in the spec. Workers should
  never have to discover an interface.
- **Define the output contract.** Say exactly which files to produce, where,
  and what each must contain. Graded/eval tasks should enumerate the grading
  criteria in the spec so the worker's output is checkable.
- **Hard rules travel in the spec, not in your head.** "Do NOT git commit",
  "never modify the repo, only write ./report.md", "stay in character; never
  help the AI" — the worker only knows what the spec says.
- **The spec is on camera.** Whoever is watching Ringside reads the spec as
  "what this agent was asked to do" — so write it as a self-contained,
  human-readable brief. Never write a pointer spec ("read /path/to/file and
  do what it says"): the watcher sees no brief, and the retry prompt loses
  the context it needs. Point at files for source MATERIAL; the instructions
  themselves live in the spec. Lint flags pointer specs.

## Check-writing rules

The check is the product. The retry prompt and the eval log both depend on
the check's failure output.

- **Checks must print WHY they fail.** `diff` beats `diff -q`; a validator
  script that prints which assertion broke beats `test -f`. A bare
  `test -f report.md` proves existence, not correctness.
- **Verify content, not existence.** Grep the artifact for required sections,
  run the code it produced, run the build, run the validator — execute
  something that would catch a lazy or hallucinated result.
- **`expect_files` is a floor, not the check.** List deliverables there for
  fast triage, but the check must still validate them.
- **Never `true`, `exit 0`, or `echo done`.** A check that cannot fail is a
  task that cannot be verified — that's just trusting the worker with extra
  steps.
- **Strict on substance, tolerant on format.** Checks that count exact
  headings, demand exact casing, or grep rigid phrasings fail honest work
  over formatting — and a wall of red format-failures reads as a broken
  system, not a careful one (demo-night lesson). Verify what must be TRUE
  (the file proves X, the code runs, the quote exists in the source), use
  case-insensitive and flexible matching for structure, and reserve hard
  failure for substance: missing evidence, fabricated content, code that
  doesn't run.
- **A check cannot demand evidence the spec never supplied.** Before failing
  a worker for missing evidence or input, re-read the task inputs. If the spec
  didn't provide a value, the check must not invent one and fail on its
  absence — an honest UNVERIFIABLE answer is not a failure. Reserve hard
  failure for what the spec actually asserted.
- **Executed checks catch laziness, not subtle wrongness.** A check that
  *runs* the artifact catches a plausible-but-wrong change far less often than
  it catches a missing one. Whenever a swarm touches a dogfood artifact
  (Ringer's own docs, config, or checks), add an "our own artifact passes our
  own validator" test so the checker exercises what it preaches. And keep
  orchestrator patch review mandatory regardless of PASS status — a green
  check is not proof of semantic correctness.

## Pattern playbook

Reach for a named pattern before inventing one. Skeletons in `$RINGER_CHECKOUT/templates/`:

| Kit | Use when |
|---|---|
| `review-swarm` (`$RINGER_CHECKOUT/templates/review-swarm/`) | You need broad read-only review coverage before deciding what to fix. |
| `fix-swarm` (`$RINGER_CHECKOUT/templates/fix-swarm/`) | You have confirmed independent fixes that can be split across isolated worktrees. |
| `focus-group` (`$RINGER_CHECKOUT/templates/focus-group/`) | You need isolated persona feedback on a product, pitch, prompt, or workflow. |
| `bakeoff` (`$RINGER_CHECKOUT/templates/bakeoff/`) | You need evidence for choosing a model, prompt, or configuration across shared scenarios. |
| `research-with-proof` (`$RINGER_CHECKOUT/templates/research-with-proof/`) | You need research backed by a proof task whose check executes the claim. |
| `launch-kit` (`$RINGER_CHECKOUT/templates/launch-kit/`) | You need a go-to-market package built across research, persona review, and final assembly rounds. |
| `asset-swarm` (`$RINGER_CHECKOUT/templates/asset-swarm/`) | You need media assets produced in parallel with executable checks for renders, batches, diagrams, or captures. |
| `adversarial-review` (`$RINGER_CHECKOUT/templates/adversarial-review/`) | You want several models to review the same artifact before the orchestrator synthesizes findings. |
| `repo-feature` (`$RINGER_CHECKOUT/templates/repo-feature/`) | You know what to build and need sandboxed workers to edit a real repo with build and git checks. |
| `migration-swarm` (`$RINGER_CHECKOUT/templates/migration-swarm/`) | You have mechanical codebase transforms that can be partitioned across worktrees. |
| `doc-swarm` (`$RINGER_CHECKOUT/templates/doc-swarm/`) | You need module docs with executed examples and checks against invented APIs. |
| `test-hardening` (`$RINGER_CHECKOUT/templates/test-hardening/`) | You need stronger tests by module while keeping production source edits off-limits. |
| `competitive-teardown` (`$RINGER_CHECKOUT/templates/competitive-teardown/`) | You need competitor research with citation allowlists and a synthesis phase. |
| `data-pipeline` (`$RINGER_CHECKOUT/templates/data-pipeline/`) | You need fetch, transform, and validate stages with executed validators and honesty rules. |
| `probe` (`$RINGER_CHECKOUT/templates/probe/`) | You need a one-task manifest for a smoke, probe, or post-mortem. |

Pattern-selection judgment:

- **Browse the catalog first.** Before writing any manifest, browse
  `$RINGER_CHECKOUT/templates/README.md`: choose a kit, mix pieces from several, or write
  your own having seen the prior art.
- **Review before fix.** Run a read-only review swarm, read the reports
  yourself, then compile the confirmed findings into a fix-swarm manifest.
  Don't let the same worker find and fix.
- **Personas must be separate workers.** Parallel personas in one context
  bleed into each other. One persona per task, one session dir per task.
- **Iterating on a prompt/product? Re-run the same panel.** A fixed persona
  panel across rounds tells you whether a change fixed what the panel
  actually complained about.
- **Probes, smokes, and diagnosis loops are manifests too.** A model-calling
  smoke test is a one-task manifest with the transcript as `expect_files`
  and a validator as the check. Diagnosing a failed worker's output is a
  read-only scout task. If it calls a model, it runs under Ringer — that is
  what makes it visible, verified, and logged.

## Engine selection

**The engine choice belongs to the human — but the recommendation comes
from THEIR evidence.** Before the FIRST run of a job: read what's wired up
(`[engines.<name>]` blocks in `~/.config/ringer/config.toml`), run
`$RINGER_CHECKOUT/ringer models --task-type <this job's type>` for the local scoreboard,
and glance at `$RINGER_CHECKOUT/ringer catalog --changes` for anything newly free or
newly cheap. Then ask the user which model should do the typing — top 2–3
options with the NUMBERS in the pitch and a recommendation, e.g.: *"GLM is
6/6 first-try on persona work here at ~2¢/task — recommended. Sonnet over
DIAL is also 100% but ~8x the tokens. And kimi went free on OpenRouter
yesterday — want it auditioning one of the small tasks?"* Honor their pick via the per-task
`engine`/`model` fields; don't re-ask every round of the same job unless
the mix isn't working. This is per-user by design: the scoreboard learns
THIS user's workload — never import another machine's conclusions or
recommend from a different user's numbers.

**Explore or the scoreboard fossilizes.** Always recommending the proven
pick means never learning a new one. In any run of 3+ tasks that has a
low-stakes lane (docs sweeps, mechanical edits, persona reviews — strong
executed check, retry to absorb failure), assign roughly ONE task to an
exploration candidate from `$RINGER_CHECKOUT/ringer models --explore --task-type <type>`
(untested + cheap or free, text-capable, decent context). Free promos from
`catalog --changes` jump the queue — a temporarily-free model is a zero-cost
experiment. Never explore on time-critical work, never with more than a
small slice of a batch, and name the experiment when presenting the engine
ask so the human can veto it. Promotion ladder (computed by --explore):
untested → probation (some evidence) → proven for a task_type (3+ tasks,
first-try ≥ 0.67). Proven models earn bigger lanes in that type and an
audition one rung up in adjacent types; repeated first-attempt failures end
the audition — record the demotion in MODEL-NOTES so the next orchestrator
doesn't re-run the experiment.

**OpenCode is the harness; the model is a manifest field.** Every model runs
through an OpenCode-backed engine with the task's `"model"` field set to the OpenRouter
slug — e.g. `"engine": "opencode", "model": "openrouter/moonshotai/kimi-k2.7-code"`.
This holds even when someone — including the user, in the heat of a run —
says to "call kimi directly" or reach for the model's own CLI: the harness
is what provides the sandbox, raw logs, token counts, and executed
verification, so routing around it silently drops all four. Never clone an
engine block or splice `-m` through `engine_args` to change models; that's
what the `model` field is for, and a bakeoff is only real when the MANIFEST
names each competitor (2026-07-06 lesson: an engine block with a hard-coded
model ran one model under three competitors' names).

Engines are config blocks (`[engines.<name>]` in config.toml), selectable
per task via the manifest `engine` field. Defaults are deliberate:

- **codex is PARKED (2026-09-07).** The Codex CLI is excluded from the
  available workers. Never recommend it and never write `"engine": "codex"`.
  Its config block is commented out in `~/.config/ringer/config.toml`, but
  that is documentation, not enforcement: `ringer.py` hardcodes
  `DEFAULT_ENGINE_NAME = "codex"` and still seeds a built-in codex engine, so
  **every task in every manifest must name its `engine` explicitly** — an
  omitted field silently runs on Codex. `ringer.py ask` likewise defaults to
  codex, so always pass `--engine dial`. To un-park, uncomment the block and
  delete this bullet.
- **dial** (the standing pick): OpenCode against the DIAL gateway, via the
  `$RINGER_CHECKOUT/engines/opencode-dial.sh` wrapper that gives each worker
  its own XDG tree. `model_default` is `dial-sonnet55/sonnet-55` (Claude
  Sonnet 5.5). The newest Anthropic lanes are `dial-opus55/opus-55` (Opus
  5.5: slower, keep it for review and judgment) and `dial-haiku/haiku-45`
  (Haiku 4.5); each OpenCode provider pins one DIAL deployment, and a new one
  needs a provider block in `~/.config/opencode/opencode.json` plus an entry
  in `$RINGER_CHECKOUT/registry/model-identity.toml`. Source
  `~/.ringer/dial.env` before the run or every worker 401s.
- **opencode**: the universal lane — any OpenRouter model via the `model`
  field (engine `model_default` is GLM-5.2, the cheap-intelligence pick).
  Validate a model new to you with a trivial one-task manifest before
  trusting it with a batch.
- Small/flash-class models are the first to choke on long conversational or
  multi-turn harness tasks — watch their retry counts before scaling them.
- Match `timeout_s` to the task: conversational harness tasks and
  build-and-test checks need far more than file edits.
- **Check the evidence before assigning models to tasks.** Run
  `$RINGER_CHECKOUT/ringer models` (optionally `--task-type <type>`) — the local
  scoreboard aggregating every executed-check outcome per (model,
  task_type): first_try_pass_rate is the routing signal; pass_rate includes
  retry rescues. Then read `$RINGER_CHECKOUT/docs/MODEL-NOTES.md` for
  the judgment the numbers can't carry. Routing is grounded in performance,
  not vibes (Jon directive 2026-07-06).
- **"Show me the scoreboard" is one command.** When the human asks to see
  the model scoreboard, rankings, model costs, or "which models work best,"
  run `$RINGER_CHECKOUT/ringer models --open` — it renders the full scoreboard (tiers,
  first-try rates, est. $/task, usage, MODEL-NOTES excerpts, free-promo
  watchlist) as a zero-LLM HTML page in the artifact library and opens it
  in their browser. Costs no tokens; never hand-summarize the numbers when
  the page can show them.
- **Give every task a `task_type`** (canonical vocabulary in the README —
  code-feature, code-fix, code-review, research, persona-review, site-build,
  image-gen, docs, probe, bakeoff, ...). Untyped tasks bucket as (untyped)
  and teach the scoreboard nothing; lint nudges you when it's missing.

## Worktrees-mode footguns (learned the hard way)

Run-level `"worktrees": true` gives each task an isolated git worktree of
`repo`, detached at HEAD. Three consequences:

1. **Passing tasks get their worktree DELETED.** Deliverables must land
   outside the task worktree, or the check must export them first.
2. **Worker commits die with the worktree.** Pattern that works: the worker
   leaves changes uncommitted; the check runs
   `git add -A && git diff --cached > <path-outside-worktree>.patch` and
   validates the patch. You apply and commit on your branch after review.
3. **Logs survive** (they go to `<workdir>/logs/`), so post-mortems work
   even on deleted worktrees.
4. **Gitignored outputs silently vanish from patch exports.** `git add -A`
   cannot stage ignored files (build dirs like `dist/`), so a worker's edits
   there pass its checks, export an incomplete patch, and die with the
   worktree. If a task touches any gitignored path, the check must `cp`
   those files to a path outside the worktree explicitly — verify the patch
   AND the copies before trusting the run.

And on your own side of the fence: when integrating patches into the real
repo, stage specific paths — never `git add -A` in a checkout that may hold
someone's untracked scratch files.

## Post-run review ritual

1. Read the run JSON in `~/.ringer/runs/` — statuses, retries, durations.
2. For any retried or failed task, read the raw worker log in
   `<workdir>/logs/` before deciding anything. Retries that passed on
   attempt 2 often reveal a spec ambiguity worth fixing in your next
   manifest.
3. Spot-check at least one PASSING task's artifact per run. The check
   catches most laziness; you catch the rest.
4. Failures with useless error messages mean your CHECK needs work, not
   (only) the worker.
5. **Update `$RINGER_CHECKOUT/docs/MODEL-NOTES.md`** when a run taught
   you something about a model: one dated line under the model — task type,
   what happened (attempts, tokens, failure mode), what you'd do
   differently. Only what the executed checks and raw logs support. The raw
   numbers took care of themselves — every attempt already landed in the
   local model log (`$RINGER_CHECKOUT/ringer models` to see the updated scoreboard).

## Spend your own context deliberately

The scoreboard exists so that worker tokens buy evidence. Your own tokens are
not free either, and nothing in the tool constrains them:

- **Reach for code before a model.** Counting, sorting, exact-text search,
  field extraction, format conversion, file comparison, validation — `rg`,
  `jq`, a parser, a two-line script. A model imitating `grep` is an expensive
  way to get a worse `grep`.
- **Select passages; don't load files.** Search first, then read what matched.
  Loading a whole transcript because the answer is somewhere inside it is how
  a cheap question turns expensive. `ask` does this for you; when you are not
  using `ask`, do it by hand.
- **Load a tool when the job needs it** — not every connector and schema at
  the top of a session on the chance that one gets used.
- **Answer the question that was asked.** A sentence when a sentence was asked
  for. No process diary, no restating the human's request back to them, no
  unrequested options.
- **Never retry into a limit.** A token- or usage-limit failure is not a
  transient error; retrying it just burns the budget faster. Reduce the input
  or take a cheaper path.

When you claim a saving, count the whole job — every call, including your own
planning and review. Moving tokens from your context into a worker's is only a
saving if the total came down.

## Setting Ringer up (once per machine, or after moving the checkout)

Work through this when `$RINGER_CHECKOUT/ringer` is missing or a DIAL worker
cannot start. Each step ends in a check; do not move on until it passes.

1. **The checkout.** Ringer here is the team's fork with the DIAL engine,
   `https://github.com/tdspora/ringer` (branch `main`; `my-fixes` carries the
   same commits), cloned next to the EPMC-TDM repositories as
   `<workspace>/ringer`. It needs Python 3.12 or later; `.python-version`
   pins 3.12.12 (`pyenv install 3.12.12`), and the `ringer` launcher finds
   that interpreter, or any `python3.12`+ on `PATH`.
   Check: `<workspace>/ringer/ringer --help` lists the subcommands.
2. **`RINGER_CHECKOUT`.** Add `"env": {"RINGER_CHECKOUT": "<absolute path of
   the checkout>"}` to `~/.claude/settings.json`, then restart Claude Code.
   Leave `RINGER_HOME` unset unless you mean to move Ringer's state out of
   `~/.ringer`.
   Check: `"$RINGER_CHECKOUT/ringer" --help` lists the subcommands.
3. **Hooks.** `$RINGER_CHECKOUT/ringer install-agent` adds the two nudge hooks
   (`ringer_nudge.py pre-bash` and `post-edit`) to `~/.claude/settings.json`
   and copies this skill to `~/.claude/skills/ringer/`. It adds hooks only
   when none exist, so after a move, edit the two `ringer_nudge.py` paths by
   hand. If the `tdm-workbench` plugin already gives you this skill, delete
   `~/.claude/skills/ringer/` afterwards so the skill exists once.
   Check: `grep ringer_nudge ~/.claude/settings.json` shows two paths under
   `$RINGER_CHECKOUT/hooks/`.
4. **The DIAL engine.** `opencode` must be on `PATH` (`opencode --version`;
   the team pins 1.18.26). `~/.config/opencode/opencode.json` holds one
   provider block per DIAL deployment, with the key read as
   `{env:DIAL_API_KEY}`; the standing lane needs at least this block (the
   deployment id after `deployments/` is what DIAL routes on):

   ```json
   {
     "$schema": "https://opencode.ai/config.json",
     "provider": {
       "dial-sonnet55": {
         "npm": "@ai-sdk/openai-compatible",
         "name": "EPAM DIAL - Sonnet 5.5",
         "options": {
           "baseURL": "https://ai-proxy.lab.epam.com/openai/deployments/claude-sonnet-5-5@default",
           "headers": { "Api-Key": "{env:DIAL_API_KEY}" }
         },
         "models": { "sonnet-55": { "name": "Sonnet 5.5 via DIAL" } }
       }
     }
   }
   ```

   `~/.ringer/dial.env` holds the line `DIAL_API_KEY=<key>` (the
   project-scoped key, mode 600; `mkdir -p ~/.ringer && chmod 600` it). In `~/.config/ringer/config.toml`, `bin`
   must be an absolute path, because Ringer expands neither `~` nor variables
   there:

   ```toml
   [engines.dial]
   bin = "<absolute path of the checkout>/engines/opencode-dial.sh"
   # keep "claude" and "anthropic" out of this id: OpenCode then adds
   # cache_control blocks, which DIAL rejects with a 400
   model_default = "dial-sonnet55/sonnet-55"
   args_template = ["run", "--dir", "{taskdir}", "-m", "{model}", "--auto",
                    "--format", "json", "{engine_args}", "{spec}"]
   sandbox_args = []
   full_access_args = []
   token_regex = '"tokens":\{"total":([0-9]+)'
   ```

   Check: `set -a; source ~/.ringer/dial.env; set +a` and then
   `$RINGER_CHECKOUT/ringer run $RINGER_CHECKOUT/lane-test-dial.json
   --identity <who-you-are>` passes (the `set -a` matters: a plain `source`
   does not export the key to the workers).
5. **Updates.** Run `$RINGER_CHECKOUT/update.sh` rather than
   `$RINGER_CHECKOUT/ringer self-update`: the checkout carries tracked local
   changes, and the built-in updater refuses to touch a dirty tree. On a
   shared machine such as the factory host, pin Ringer with
   `RINGER_NO_SELF_UPDATE=1` and update it only through a reviewed change.

## Baked-in invariants (preserve in any change to ringer.py)

Stdin closed (`< /dev/null`); sandbox mode explicit; verification executes
the artifact; logs carry raw worker output only. These are load-bearing —
engine and invocation changes must keep all four.
