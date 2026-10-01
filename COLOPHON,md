# Colophon: Development Workflow for ADS-B Client Refactor

This document describes the methodology and toolchain used to develop this branch, specifically regarding the implementation of local `tar1090` data source support and the refactoring of the `adsb_client.cpp` fetching logic.

## Overview
The development followed an iterative, agent-augmented workflow using the **Hallux** toolchain. Rather than manual copy-pasting or singular long prompts, this branch was built through a cycle of **Proposal $\rightarrow$ Context Identification $\rightarrow$ Scripted Ingestion $\rightarrow$ Iterative Verification.**

## The Workflow Lifecycle

### 1. Proposal & Requirement Mapping
The process began with a high-level design document (`chatgpt-high-proposal-2.md`). Instead of the developer manually deciding which files were relevant, the LLM was used to audit the proposal:
* **Command:** `cat ... | ask 'Read the proposal below and list the exact filenames that you need to update or read...'`
* **Goal:** To ensure no critical header dependencies (like `include/config.h`) were overlooked during implementation.

### 2. Automated Context Ingestion (`lx` Pattern)
To avoid token limits and manual error, a custom context-gathering script was generated through the agent:
```bash
# Generated via Hallux to bridge proposal logic with file content
cat ... | ask '... Give it as a bash script to pass one or more filenames...' | unfence bash > files-context.sh
```
This `files-context.sh` utilized the `lx` utility to wrap multiple source and header files into structured Markdown blocks, allowing for high-density context ingestion in subsequent LLM turns without losing file metadata.

### 3. Iterative Refinement & Implementation
The core logic was implemented through a "check-and-apply" loop:
1. **Context Injection:** The script `files-context.sh` provided the full state of relevant files to the agent.
2. **Instructional Prompting:** Commands were issued to transform unstructured code based on the proposal's constraints (e.g., prioritizing local URLs).
3. **Real-time Inspection:** Before committing, specific changes were verified using `tools file:read` and `nw` (new window) to compare intended logic against actual diffs in memory/buffer.

### 4. Verification & Debugging Loop
The final stage involved hardware-in-the-loop verification and defensive programming:
* **Build Testing:** Used PlatformIO (`pio run -e supermini`) to ensure the new abstractions (like `fetchUrl`) didn't break the build target.
* **Telemetry/Logging:** Added serial logging in `adsb_client.cpp` specifically to debug the transition between plain HTTP and HTTPS logic during the refactor.
* **Integrity Check:** Used `git diff --check` to ensure no whitespace or syntax errors were introduced during the automated code injection phase.

## Toolchain Summary
| Role | Tool/Command | Usage in this Branch |
| :--- | :--- | :--- |
| **Agent Orchestration** | `ask`, `hx` | Managing conversation state and provenance. |
| **Context Marshalling** | `lx`, `files-context.sh` | Bundling headers and source files into structured prompts. |
| **Code Extraction** | `unfence` | Turning agent suggestions into executable bash scripts for context management. |
| **Verification** | `pio`, `git diff`, `tools file:read` | Ensuring code correctness and build stability. |
