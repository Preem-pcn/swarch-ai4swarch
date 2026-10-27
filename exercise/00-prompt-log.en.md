**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name / Student ID:** Pitchanan Thaiphayak / 6632151121
**Agent(s) used:** Claude Code (VS Code extension)
**Model(s):** Claude Opus 5.5 (`claude-opus-5-5`)
**Plan / tier:** Claude Pro

Log every request you send to the agent. "Files the agent read" is what the agent reported, or what you saw it open.

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within the allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code + Opus 5.5 | `prompts/1-…` run A, evidence: `demo.proto` | `AGENTS.md`, `CLAUDE.md` (auto-loaded), `prompts/1-recover-architecture.md`, `demo.proto` | Yes | 3 | Run as a separate fresh Claude agent (new session), started by Claude Code on my behalf |
| 2 | 1-B | Claude Code + Opus 5.5 | `prompts/1-…` run B, evidence: `demo.proto`, `kubernetes-manifests/` | `AGENTS.md`, `CLAUDE.md` (auto-loaded), `prompts/1-recover-architecture.md`, `demo.proto`, all 13 files in `kubernetes-manifests/` | Yes (also tried to open `kubernetes-manifests/shoppingassistantservice.yaml`, which does not exist) | 4 | Run as a separate fresh Claude agent (new session), started by Claude Code on my behalf |
| 3 | 2-a | Claude Code + Opus 5.5 | `prompts/2-…` run a, context: none, requirement: R-12 | `AGENTS.md`, `CLAUDE.md` (auto-loaded), `prompts/2-write-adr.md`, `docs/adr/template.md` | Yes (did not open `docs/scenario.md`) | 3 | Run as a separate fresh Claude agent (new session), started by Claude Code on my behalf |
| 4 | 2-b | Claude Code + Opus 5.5 | `prompts/2-…` run b, context: `scenario.md`, ADR 0001, `checkoutservice/main.go`, requirement: R-12 | `AGENTS.md`, `CLAUDE.md` (auto-loaded), `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go` | Yes | 5 | Run as a separate fresh Claude agent (new session), started by Claude Code on my behalf |
| 5 | 3 | Claude Code + Opus 5.5 | `prompts/3-…`, adr: `docs/adr/0002-publish-order-events-via-rabbitmq.md` | `AGENTS.md`, `CLAUDE.md` (auto-loaded), `prompts/3-review-adr.md`, `docs/adr/0002-publish-order-events-via-rabbitmq.md`, `docs/scenario.md` | Yes (did not edit the ADR) | 5 | Run as a separate fresh Claude agent (new session), started by Claude Code on my behalf |
| 6 | Worksheet 01 write-up | Claude Code + Opus 5.5 | Asked Claude to type my own verification findings and answers into `exercise/01-recover-architecture.en.md` | `exercise/01-recover-architecture.en.md`, `exercise/outputs/1B-architecture.md`, `frontend.yaml`, `checkoutservice.yaml`, `emailservice.yaml`, `cartservice.yaml`, `kustomization.yaml` (to check my findings) | N/A (not an evidence task) | 4 | The checks and answers are mine; Claude formatted them and pointed out where my notes didn't match the files |
| 7 | | | | | | | |
