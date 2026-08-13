# Full-duplex attempts

An atlas of my attempts at building a full-duplex voice system — a system where
the user and the assistant can both speak at once, and each hears the other while
speaking.

This repository contains **no code**. Every attempt lives in its own repository.
What lives here is the part that never survives inside a single project: what each
attempt was actually trying, which half of the problem it solved, where it hit a
wall, and what the wall was made of.

## The attempts

| | [Assistant mark I](docs/attempts/assistant-mark-i.md) | [voxtral-live](docs/attempts/voxtral-live.md) |
|---|---|---|
| Repository | [Brightwav3/Jarvis](https://github.com/Brightwav3) (superproject, 11 submodules) | [Brightwav3/voxtral-live](https://github.com/Brightwav3/voxtral-live) |
| Language | TypeScript | JavaScript (ESM) |
| Architecture | Native speech-to-speech (Gemini Live) | Cascade: STT → chat → TTS (Mistral) |
| Generation | **2** — one model, still turn-based | **1** — three models in series |
| Turn detection | Provider-side, still on silence | 550 ms silence timer, in-process |
| Half solved | **Media** — audio genuinely flows in and out of one model | **Application** — delegation, cancellation, barge-in plumbing |
| Missing | Tool delegation from the realtime path; echo cancellation | The middle: the cascade caps how duplex it can ever be |
| Duplex rating | Capped by the model, ready for a better one | Capped by the architecture |

Neither attempt is generation 3, and neither could have been:
[no obtainable full-duplex model speaks Czech, and none is hosted](docs/the-model-problem.md).
That is the ceiling both hit, and it is worth separating from the walls each one
built for itself.

The one-line version: **the two projects are complementary halves of the same
architecture.** Neither is a failed attempt; each is the other's reference
implementation for the half it lacks. That relationship is the main thing this
repository exists to record.

## Read in this order

1. [What full-duplex actually requires](docs/what-full-duplex-requires.md) — the
   five criteria every attempt below is judged against, and why "just remove the
   VAD" is not one of them.
2. [The model problem](docs/the-model-problem.md) — which full-duplex models
   exist, what they cost, and why the list is still this short. Criterion 4 is
   the only one you cannot build your way past, so it is worth knowing how tall
   that wall is before reading how each attempt hit it.
3. [Assistant mark I](docs/attempts/assistant-mark-i.md) — the media half.
4. [voxtral-live](docs/attempts/voxtral-live.md) — the application half.
5. [What I learned](docs/lessons.md) — the findings that outlived the code.
6. [Glossary](docs/glossary.md) — barge-in, AEC, cascade, turn-taking, and the
   rest, defined once so the attempt pages can be terse.

## Ground rules for this repo

- **Claims cite code.** Every architectural claim on an attempt page names the
  file and construct it came from, so a future reader can check whether it is
  still true rather than trusting a summary.
- **Findings are dated.** Repositories move. A page describes what was true on
  the date at its top, against the commit named there.
- **No vendored code, no submodules.** Links only. A pinned SHA that silently
  rots is worse than a link that is obviously stale.
