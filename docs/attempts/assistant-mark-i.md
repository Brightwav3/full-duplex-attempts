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
| 2. Echo cancellation | **None, and nowhere to put it.** See below. |
| 3. No blocking turn detector | **Yes.** Turn detection is the provider's problem, not a timer in this codebase. |
| 4. Native speech-to-speech | **Yes.** This is the attempt's real achievement. |
| 5. Interruptibility | **Partial.** `interrupt()` and `output.interrupted` exist on the contract; there is no delegated work to cancel, because there is no delegation. |

## The two walls

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

## What this attempt proves

That the hard half — a native speech-to-speech session behind a
provider-independent contract, with a fake that makes it testable — is buildable
and was built. What it did not prove is that a duplex conversation is useful
without the ability to act, or that module independence survives contact with
acoustics.

The pattern for the missing half does not need inventing. See
[voxtral-live](voxtral-live.md), whose `delegation.mjs` and `cancellation.mjs`
are a working reference for dispatch-not-await with cancellation on supersede.
