# Assistant mark I

> **Read as of:** 2026-08-13, against superproject commit
> [`63105a2`](https://github.com/Brightwav3/Assistant-mark-I/commit/63105a2)
> in [Brightwav3/Assistant-mark-I](https://github.com/Brightwav3/Assistant-mark-I)
> (branch `codex/realtime-tools-audio-boundary`, 11 submodules).
> **Repository:** [github.com/Brightwav3](https://github.com/Brightwav3) — a
> superproject; each core is an independent repository.
> **Working name:** "Jarvis" appears in the prose and never in the code; every
> package is named for what it does.

## What it was trying

Infrastructure for a persistent, ambient, **model-independent** assistant. The
governing constraint, from the manifesto: a better model should drop into the
same assistant without requiring surgery on the rest of the system. Hence the
submodule-per-core layout, and hence contracts (`RealtimeSpeechProvider`,
`RealtimeSpeechSession`) that no provider's vocabulary leaks through.

Full-duplex is not the project's headline goal. It is the thing the speech path
was built to make *possible*, which is why the attempt is worth recording here.

## Architecture, in one paragraph

Eleven cores as git submodules: `core-runtime`, `memory-core`, `state-core`,
`intelligence-core`, `device-network`, `activation-core`, `tool-system`,
`host-tools`, and `speech-system`, which itself contains three:

- **Realtime Core** — holds a live session with a native speech-to-speech
  provider. Audio in, audio out, one model.
- **Scribe Core** — capture.
- **Voice Core** — the local voice surface.

Realtime Core ships two providers behind one interface: `gemini.ts`
(`GeminiLiveSession`, a genuine Gemini Live session) and `fake.ts` (a
deterministic stand-in for tests). Provider selection is config, and the fake
implements the same contract as the real one — which is why the test suite can
cover the session lifecycle without touching a network.

## Against the criteria

| Criterion | Status |
|---|---|
| 1. Simultaneous I/O | **Yes, structurally.** The Gemini Live session is inherently bidirectional. |
| 2. Echo cancellation | **None, and nowhere to put it.** See Wall 1. |
| 3. No blocking turn detector | **No — outsourced, not removed.** See Wall 2. |
| 4. Full-duplex model | **No. Generation 2.** Gemini Live is native speech-to-speech and still turn-based. |
| 5. Interruptibility | **Partial but useful.** `interrupt()` and `output.interrupted` stop stale playback; realtime tool execution propagates cancellation and drops late results. It is not generation-3 conversational overlap. |

> **Correction, 2026-08-13.** An earlier version of this page scored criteria 3
> and 4 as passes, on the reasoning that turn detection is the provider's problem
> and that Gemini Live is a native speech-to-speech model. The second half is
> true and the conclusion does not follow: native speech-to-speech is
> [generation 2](../what-full-duplex-requires.md), and generation 2 still ends
> the turn on silence — server-side, but no less imposed. The attempt's real
> achievement is the contract, not the duplexity.

## The remaining walls

### Resolved engineering gap — tools now reach the path that runs on hardware

The previous version of this page called tool access unreachable. Mark I now
closes that gap with a provider-neutral boundary:

```ts
export interface RealtimeSessionConfig {
  provider: string; inputFormat: AudioFormat; model?: string;
  systemInstruction?: string; apiKey?: string; tools?: RealtimeToolDeclaration[];
}
```

`RealtimeSpeechEvent` now has `tool.requested`, and
`RealtimeSpeechSession.sendToolResult()` completes the return path. The
`assistant-runtime/src/tool-bridge.ts` adapter translates Tool System
declarations and outcomes without duplicating validation or policy. The normal
composition exposes four safe read-only tools — `calculate`, `get_time`,
`system_status`, and `uptime` — and the 2026-08-13 hardware run confirmed real
calls for time, system status, and multi-step calculation.

Side-effecting capabilities such as `open_app` are still deliberately explicit:
they require an injected executor, an allowlist policy, and a broker. That is a
permission boundary, not a missing integration.

### Wall 1 — echo cancellation has no home

Capture (Scribe Core) and playback (Realtime Core) are **separate repositories
with zero imports between them, by design**. Module independence is the project's
central value, and here it collides with a physical requirement: AEC needs one
component holding both signals on one timeline.

This is the honest justification for a coordinating component — not "coordination
of conversational flow", which is vague enough to justify anything, but a
concrete signal-processing requirement that the current decomposition makes
unimplementable.

### Wall 2 — the ceiling is the model, and the model is generation 2

The frustration this attempt produced was that it did not feel like GPT‑Live,
despite the architecture being right. The cause is not in the repository. Gemini
Live is a turn-based native model: it waits for the user to stop before it
starts, and it cannot backchannel, because emitting anything means the turn
changed hands. No contract, no state model, and no amount of application work
closes that gap. It is closed by a different model or not at all.

This is the wall worth naming most precisely, because it is the one that looks
like a personal failure and is not. See
[The model problem](../the-model-problem.md) for what is obtainable, at what
price, and why it is still thin.

The corollary is unflattering in a different way. Because the ceiling is
external, the remaining internal wall is AEC placement; tool access is now part
of the verified Mark I boundary rather than an open gap.

## What this attempt proves

That the hard half — a native speech-to-speech session behind a
provider-independent contract, with a fake that makes it testable, plus a
provider-neutral Tool System bridge and safe hardware-verified capabilities — is
buildable and was built. It is generation-2 infrastructure built well enough
that generation 3 should drop into it as a provider rather than a rewrite;
whether that is true is untested, and stays untested until a second provider
ships.

What it did not prove is that module independence survives contact with
acoustics, or that a generation-3 model will preserve the same contracts under
continuous overlap.

The pattern for the remaining application-side improvements does not need
inventing. See [voxtral-live](voxtral-live.md), whose `delegation.mjs`,
`cancellation.mjs`, and echo-suppression path are working references for the
parts Mark I does not yet own.
