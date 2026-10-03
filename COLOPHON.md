# Colophon: Development Workflow for ESP32-Plane-Radar Enhancements

This document describes the methodology and toolchain used to develop this branch. It covers two major efforts: (A) the initial local `tar1090` data-source integration and `adsb_client` refactor, and (B) the subsequent aircraft-display enhancement (altitude coloring, zoom-dependent label visibility, and compact tag layout) implemented as four incremental sub-tasks.

Specification development was done through OpenAI ChatGPT and specification documents were provided to Hallux.

All work was performed through the **Hallux** shell-agent toolchain, operating as a Unix pipeline with `stdin`/`stdout` interaction, structured context ingestion, and direct file-mutation tooling.

## Specification Reference 
https://chatgpt.com/share/6ac18280-a734-83e8-b8af-a2ae21fa2db8

---

## A. tar1090 Data-Source Refactor (Phase 1)

### A.1 Proposal & Requirement Mapping

The process began with a high-level design document (`chatgpt-high-proposal-2.md`). Rather than the developer manually deciding which files were relevant, the LLM was used to audit the proposal:

```bash
cat .hallux/tar1090/chatgpt-high-proposal-2.md | ask \
  'Read the proposal below and list the exact filenames that you need to update or read.
   I will give them back to you in an archive later.
   Give it as a bash script to pass one or more filenames to one or more calls to `lx`.' \
  | tools file:read \
  | unfence bash > .hallux/tar1090/files-context.sh
```

* **Goal:** Ensure no critical header dependencies (e.g., `include/config.h`) were overlooked.
* **Output:** A reusable `files-context.sh` that bundles the identified files via `lx`.

### A.2 Automated Context Ingestion (`lx` Pattern)

The generated script wraps multiple source and header files into structured Markdown blocks (filename header + quad-backtick content + `---` terminator), allowing high-density context injection without manual copy-paste:

```bash
bash .hallux/tar1090/files-context.sh | ask \
  'output the updated files according to the proposal.
   if you have insufficient information, first say so and then stop.'
```

### A.3 Iterative Verification

```bash
bash .hallux/tar1090/files-context.sh | ask \
  'check that the required changes have been made according to the proposal' \
  | tools file:read
```

Followed by hardware-in-the-loop build testing (`pio run -e supermini`) and `git diff --check`.

---

## B. Aircraft-Display Enhancement (Phase 2)

Hallux worked best with a ChatGPT-produced 4-step incremental plan, each step scoped to a single concern with explicit file-allowlists, required semantics, verification checkpoints, and a hard **STOP** boundary:

| Step | Concern | Files Changed |
|:---:|:---|:---|
| 1 | Numeric altitude in `Aircraft` model | `adsb_client.h`, `adsb_client.cpp` |
| 2 | Altitude-based RGB565 marker coloring | `radar_theme.h`, `radar_display.cpp` |
| 3 | Zoom-dependent tag-field visibility | `radar_theme.h`, `radar_display.cpp` |
| 4 | Compact tag layout (no blank lines) | `radar_display.cpp` |

Each step's spec (`chatgpt-plan-2-step-N.md`) includes:

- **Prerequisites** (what prior steps guarantee)
- **Goal** (one sentence)
- **Files allowed to change** (whitelist)
- **Required changes** (numbered, with exact field names and semantics)
- **Prohibited changes** (explicit "Do not modify" list)
- **Verification checkpoint** (`git diff --check`, scoped `git diff --`, `pio run -e supermini`)
- **STOP** instruction

### B.3 Per-Step Execution Pattern

Each step followed the same two-phase pipeline. The context subshell sources `files-context.sh` (which `lx`-wraps all relevant source files), appends the overview and the specific step spec, then pipes through `ask` twice:

**Phase 1 – Clarity check:**

```bash
(.hallux/tar1090-color-steps/files-context.sh \
 .hallux/tar1090-color-steps/chatgpt-plan-2-step-1234-overview.md \
 .hallux/tar1090-color-steps/chatgpt-plan-2-step-2.md ; \
 cat .hallux/tar1090-color-steps/chatgpt-plan-2-step-2.md) | ask \
 'step 1 is done. is the step-2 proposal clear'
```

**Phase 2 – Implementation with file tools:**

```bash
(.hallux/tar1090-color-steps/files-context.sh \
 .hallux/tar1090-color-steps/chatgpt-plan-2-step-1234-overview.md \
 .hallux/tar1090-color-steps/chatgpt-plan-2-step-2.md ; \
 cat .hallux/tar1090-color-steps/chatgpt-plan-2-step-2.md) | ask \
 'step 1 is done. is the step-2 proposal clear' \
 | ask \
 'read files necessary one by one and implement the proposal with concrete changes' \
 | tools file:read:write --log-level=DEBUG \
 | tee .hallux/tar1090-color-steps/step-2-impl-1.md
```

Key pipeline elements per step:

| Element | Purpose |
|:---|:---|
| `files-context.sh` in subshell | `lx`-wraps all relevant `.h`/`.cpp` files with filename headers |
| `overview.md` + `step-N.md` via `cat` | Injects the project summary and the specific sub-task spec |
| First `ask` | Clarity / readiness gate before implementation |
| Second `ask` | Instructs the agent to use file tools for direct mutation |
| `tools file:read:write` | Enables the agent to read existing files and write modified content in-place |
| `--log-level=DEBUG` | Exposes tool-call trace on `stderr` for debugging |
| `tee` / `answer > file` | Captures the full agent response (including tool calls) to a step-impl log |

Step-specific variations:

- **Step 1:** Used `tools file` (read-only) → `answer > step-1-impl-1.md`
- **Step 2:** Used `tools file:read:write --log-level=DEBUG` → `tee step-2-impl-1.md`
- **Step 3:** Same as Step 2 → `tee step-3-impl-1.md`
- **Step 4:** Same as Step 2 → `answer -t > step-4-impl-1.md` (tee via `answer -t`)

### B.4 Verification Checkpoints

After each step's implementation, the verification commands specified in the step file were run:

```bash
git diff --check
git diff -- include/services/adsb_client.h src/services/adsb_client.cpp   # step-specific files
pio run -e supermini
```

Build output (RAM/flash usage, warnings) was reported before proceeding to the next step.

### B.5 Colophon Self-Generation

This colophon is itself generated through the Hallux pipeline, using `bx` (block-execute) to inline the HALLUX.md spec, the prior colophon, and the bash history as context:

```bash
(bx cat ~/wip/answer/HALLUX.md; \
 lx .hallux/tar1090-color-steps/chatgpt-plan-2-step-1234-overview.md \
    .hallux/tar1090-color-steps/chatgpt-plan-2-step-{1,2,3,4}.md; \
 bx cat COLOPHON.md; \
 bx grep ask .hallux/.bash_history/*) | ask \
 'Output an updated COLOPHON.md based on the overview and step files
  generated by ChatGPT and the bash history.' \
 | answer > COLOPHON-2.md
```

---

## Toolchain Summary

| Role | Tool / Command | Usage in This Branch |
|:---|:---|:---|
| **Agent Orchestration** | `ask`, `hx`, `answer` | Multi-turn conversation, tool enablement, output formatting (`--json`, `-t`, `--answer`) |
| **Context Marshalling** | `lx`, `files-context.sh` | Bundling headers and source files into structured Markdown blocks for pipeline ingestion |
| **Command-Output Context** | `bx` | Inlining the output of shell commands (git, cat, grep) as structured blocks within a prompt |
| **Code Extraction** | `unfence` | Extracting executable bash from agent-generated code fences (used to create `files-context.sh`) |
| **File Mutation** | `tools file:read:write` | In-place file reading and writing during implementation steps |
| **Self-Referential Context** | `bx cat HALLUX.md`, `bx cat COLOPHON.md` | Feeding the agent its own operational spec and prior colophon for meta-tasks |
| **Verification** | `pio run -e supermini`, `git diff --check`, `git diff --stat` | Build correctness, whitespace integrity, scoped change inspection |
| **Provenance / Logging** | `tee`, `answer >`, `--log-level=DEBUG` | Capturing full agent transcripts per step for audit |

---

## Key Lessons Encoded in the Workflow

1. **Monolithic plans fail; incremental steps succeed.** The plan-1 single-shot attempt produced incoherent output. The plan-2 four-step decomposition, with per-step file allowlists and STOP boundaries, produced clean, compilable results.
2. **Clarity gate before implementation.** Each step's first `ask` call verifies the spec is unambiguous before the agent touches files.
3. **Explicit "do not change" lists prevent scope creep.** Every step file enumerates what must *not* be modified (transport code, display geometry, task setup, etc.).
4. **`lx`/`bx` context wrapping replaces manual file pasting.** The `files-context.sh` script and `bx` blocks ensure the agent sees file boundaries and command provenance without token-limit issues.
5. **Verification is codified in the spec.** Each step's "Verification checkpoint" section specifies the exact `git diff` scoping and `pio` target, making the check reproducible and auditable.
