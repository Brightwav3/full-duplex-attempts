# voxtral-live

> **Read as of:** 2026-08-13, against `origin/main` at commit `9fe7269`
> ("fix(voice): make live conversation work end to end").
> **Repository:** [github.com/Brightwav3/voxtral-live](https://github.com/Brightwav3/voxtral-live)
> **Caution:** the local worktree at `C:\Users\Sajmon\.codex\worktrees\voxtral-live-publish`
> is stale by roughly twenty commits. Read `origin/main`, never the README.

## What it was trying

A headless, always-running voice daemon on Mistral models — Voxtral for
transcription, Mistral chat, Mistral TTS — installable as a Windows service
([ADR-001](https://github.com/Brightwav3/voxtral-live/blob/main/docs/decisions/ADR-001-headless-background-daemon.md)),
driven over a control IPC channel, with a CLI on the other end.

Ambition was deliberately narrower than Assistant mark I: one provider, one
process, no module independence. That narrowness is why the *application* layer
got further here than anywhere else.

## Architecture, in one paragraph

A cascade. `src/providers/mistral-realtime-stt.mjs` streams transcription;
`src/conversation/turn-controller.mjs` decides when an utterance ended;
`src/providers/mistral-chat.mjs` generates; `src/conversation/sentence-chunker.mjs`
splits the reply so TTS can start before generation finishes;
`src/providers/mistral-tts-stream.mjs` and `src/audio/playback-queue.mjs` speak it.
Audio I/O is PortAudio (`src/audio/portaudio-backend.mjs`). Around all of it,
`src/conversation/cancellation.mjs` tracks turn and generation ids, and
`src/conversation/delegation.mjs` runs tool work off the critical path.

Test coverage is real and per-module — nineteen files under `test/`, including
`turn-controller`, `echo-suppressor`, `delegation`, `session`, and `daemon-contract`.

## Against the criteria

| Criterion | Status |
|---|---|
| 1. Simultaneous I/O | **Accidentally yes.** `handleAudioFrame` processes microphone input during `SPEAKING` — but the state model says that cannot happen. |
| 2. Echo cancellation | **Detector, not canceller.** Fails on the case that matters. |
| 3. No blocking turn detector | **No.** A 550 ms silence timer sits in the audio path. |
| 4. Native speech-to-speech | **No.** Cascade. This is the cap. |
| 5. Interruptibility | **Yes — the best part of the project.** |

## The four findings

### 1. Partials are collected and thrown away

`turn-controller.mjs` has `pushPartial()` as a **deliberate no-op**. The reasoning
is right — a partial transcript must not *start a turn*, or every hesitation
becomes an interruption. The overcorrection is that partials then have no effect
at all, so nothing can be shown or acted on until the silence timer fires.

The fix is to separate authority from visibility: keep `history` as the
authoritative record, add a `provisional` string, publish `user_transcript` with
`final: false`. Cheap on its own, and a prerequisite for finding 3.

### 2. One state enum implies an exclusion that does not hold

`src/conversation/session.mjs` models the session as `IDLE / LISTENING /
THINKING / SPEAKING` — a single variable, so the states are mutually exclusive by
construction. But `handleAudioFrame` already processes microphone input while the
session is `SPEAKING`.

So the code does the duplex thing and the model denies it. Every question of the
form "what should happen when the user talks over the assistant?" has to be
answered outside the state machine, because the state machine has no state for
it. Splitting into two independent variables — input state and output state —
makes *listening while speaking* an expressible state rather than an accident.

**This is the most transferable finding in the repository.** A single enum is the
default way to model a conversation, and it quietly encodes half-duplex.

### 3. The 550 ms detector is the architecture, not a bug

`createTurnController({ silenceMs = 550 })` is precisely the component that
full-duplex architectures remove. A cascade cannot remove it — the chat model
needs a complete utterance. What it can do is make the guess cheap: start
generation on a stable partial, cancel when more speech arrives. The machinery
already exists in `cancellation.mjs`, where `begin()` supersedes prior work by
id. Cost is wasted tokens; benefit is 550 ms off perceived latency on every turn.

### 4. The echo suppressor drops real barge-ins

`src/audio/echo-suppressor.mjs` is a detector: `isPlaybackEcho()` returns a
boolean and the frame is dropped, above a correlation threshold of `0.82`. In
speaker mode, when the user speaks over the assistant, voice **plus** echo can
still correlate above the threshold — so the genuine interruption is discarded.
The component fails exactly on the case it exists to handle.

The fix reuses the lag search already there: remember the best offset, subtract
the scaled reference, treat residual RMS above a threshold as user speech. A
mini-AEC, and strictly better than the boolean even once real AEC lands.

## What it got right, and Assistant mark I did not

**Asynchronous delegation with an immediate spoken acknowledgement, and stale
work discarded by id.** Verified end to end: `daemon.mjs:79` passes
`tools: [WEB_SEARCH_TOOL]`; `session.mjs:193` catches the `tool_call`;
`delegation.mjs` runs it and drops results belonging to a superseded turn.

The assistant says "let me look that up" *now* and delivers *later*, and if the
user has moved on by then, the answer is dropped rather than spoken into a
different conversation. That is the correct shape, and it is the shape Assistant
mark I's realtime session cannot express at all.

*Operational note:* if web search appears broken, suspect a missing
`VOXTRAL_SEARCH_ENDPOINT` environment variable before suspecting the code.

## What this attempt proves

That the application layer of a duplex assistant — delegation, supersede
semantics, chunked speech, a service that survives a reboot — is solvable, and
that solving it does not make a cascade full-duplex. The cap is criterion 4, and
no amount of work above it moves the ceiling.
