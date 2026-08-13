# Assistant mark I

> **Read as of:** 2026-08-13, against the working tree at `C:\Users\Sajmon\Jarvis`
> (superproject `main`, 11 submodules).
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
| 2. Echo cancellation | **None, and nowhere to put it.** See wall 2. |
| 3. No blocking turn detector | **No — outsourced, not removed.** See wall 3. |
| 4. Full-duplex model | **No. Generation 2.** Gemini Live is native speech-to-speech and still turn-based. |
| 5. Interruptibility | **Partial.** `interrupt()` and `output.interrupted` exist on the contract; there is no delegated work to cancel, because there is no delegation. |

> **Correction, 2026-08-13.** An earlier version of this page scored criteria 3
> and 4 as passes, on the reasoning that turn detection is the provider's problem
> and that Gemini Live is a native speech-to-speech model. The second half is
> true and the conclusion does not follow: native speech-to-speech is
> [generation 2](../what-full-duplex-requires.md), and generation 2 still ends
> the turn on silence — server-side, but no less imposed. The attempt's real
> achievement is the contract, not the duplexity.

## The three walls

### Wall 1 — tools are unreachable from the path that runs on hardware

`speech-system/realtime core/src/contracts.ts` is the whole story in two lines:

```ts
export interface RealtimeSessionConfig {
  provider: string; inputFormat: AudioFormat; model?: string;
  systemInstruction?: string; apiKey?: string;
}
```

No `tools` field. And `RealtimeSpeechEvent` — eleven variants covering session
lifecycle, input speech start/end, partial and final transcripts for both
directions, output audio start/chunk/complete, and interruption — has **no
tool-call variant**.

The consequence: the entire Tool System and the Host Tools catalogue are
unreachable from the realtime session. Not "not yet integrated" — not
expressible. The assistant can converse and cannot act.

The return path is already half built. `RealtimeSpeechSession` declares
`sendText(text: string)`, and `gemini.ts:30` implements it as
`this.#client.sendRealtimeInput({ text })` — so a tool result has a way back
into the conversation the moment there is a way for a tool call to come out.

### Wall 2 — echo cancellation has no home

Capture (Scribe Core) and playback (Realtime Core) are **separate repositories
with zero imports between them, by design**. Module independence is the project's
central value, and here it collides with a physical requirement: AEC needs one
component holding both signals on one timeline.

This is the honest justification for a coordinating component — not "coordination
of conversational flow", which is vague enough to justify anything, but a
concrete signal-processing requirement that the current decomposition makes
unimplementable.

### Wall 3 — the ceiling is the model, and the model is generation 2

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
external, the walls that *are* internal — no tool path, nowhere for AEC — are the
whole of what this attempt controls, and both remain open.

## What this attempt proves

That the hard half — a native speech-to-speech session behind a
provider-independent contract, with a fake that makes it testable — is buildable
and was built. It is generation-2 infrastructure built well enough that
generation 3 should drop into it as a provider rather than a rewrite; whether
that is true is untested, and stays untested until a second provider ships.

What it did not prove is that a duplex conversation is useful without the ability
to act, or that module independence survives contact with acoustics.

The pattern for the missing half does not need inventing. See
[voxtral-live](voxtral-live.md), whose `delegation.mjs` and `cancellation.mjs`
are a working reference for dispatch-not-await with cancellation on supersede.
