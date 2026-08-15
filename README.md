# Full-duplex attempts

An atlas of my attempts at building a full-duplex voice system — a system where
the user and the assistant can both speak at once, and each hears the other while
speaking.

This repository contains **no code**. Every attempt lives in its own repository.
What lives here is the part that never survives inside a single project: what each
attempt was actually trying, which half of the problem it solved, where it hit a
wall, and what the wall was made of.

## The attempts

| | [voxtral-live](docs/attempts/voxtral-live.md) | [Assistant mark I](docs/attempts/assistant-mark-i.md) | [Assistant mark II](docs/attempts/assistant-mark-ii.md) |
|---|---|---|---|
| Repository | [Brightwav3/voxtral-live](https://github.com/Brightwav3/voxtral-live) | [Brightwav3/Assistant-mark-I](https://github.com/Brightwav3/Assistant-mark-I) (superproject, 11 submodules) | [Brightwav3/Assistant-mark-II](https://github.com/Brightwav3/Assistant-mark-II) (superproject, independent cores) |
| Language | JavaScript (ESM) | TypeScript | TypeScript |
| Architecture | Cascade: STT → chat → TTS (Mistral) | Native speech-to-speech (Gemini Live) | Native voice frontend → delegated text model → Tool System / Memory Core → same session |
| Generation | **1** — three models in series | **2** — one model, still turn-based | **2** — one voice model plus a separate background model, still turn-based |
| Turn detection | 550 ms silence timer, in-process | Provider-side, still on silence | Provider-side, still on silence |
| Half solved | **Application** — delegation, cancellation, barge-in plumbing, and echo suppression | **Media + bounded capability execution** — audio genuinely flows in and out of one model, and safe tools reach the platform | **Delegated capability** — immediate acknowledgement, background reasoning, bounded memory tools, structured results, and same-session delivery |
| Missing | The middle: the cascade caps how duplex it can ever be | Echo cancellation and a generation-3 model; side-effecting tools are deliberately opt-in | A generation-3 model, verified overlapping speech, and device-independent AEC; provider-specific scheduling remains bounded |
| Duplex rating | Capped by the architecture | Capped by the model, ready for a better one | Capped by the model; the delegation wall is solved, not the duplex wall |

None of the attempts is generation 3, and none could have been:
[no obtainable full-duplex model speaks Czech, and none is hosted](docs/the-model-problem.md).
That is the ceiling they hit, and it is worth separating from the walls each one
built for itself.

The one-line version: **the three projects solve different internal walls around
the same problem.** Mark I has the native realtime media path and a provider-
neutral Tool System bridge; Mark II adds a verified delegated intelligence path
to that native session; voxtral-live has the richer cascade-side cancellation
and echo work. None is generation 3, and none is a failed attempt.

## Read in this order — generation 1 → generation 2

1. [What full-duplex actually requires](docs/what-full-duplex-requires.md) — the
   five criteria every attempt below is judged against, and why "just remove the
   VAD" is not one of them.
2. [The model problem](docs/the-model-problem.md) — which full-duplex models
   exist, what they cost, and why the list is still this short. Criterion 4 is
   the only one you cannot build your way past, so it is worth knowing how tall
   that wall is before reading how each attempt hit it.
3. [voxtral-live](docs/attempts/voxtral-live.md) — generation-1 cascade with
   cancellation, delegation, and echo work.
4. [Assistant mark I](docs/attempts/assistant-mark-i.md) — generation-2 native
   media plus a bounded safe tool path.
5. [Assistant mark II](docs/attempts/assistant-mark-ii.md) — generation-2
   delegated voice intelligence, same-session delivery, and the boundary between
   capability and full duplex.
6. [What I learned](docs/lessons.md) — the findings that outlived the code.
7. [Glossary](docs/glossary.md) — barge-in, AEC, cascade, turn-taking, and the
   rest, defined once so the attempt pages can be terse.

## Ground rules for this repo

- **Claims cite code.** Every architectural claim on an attempt page names the
  file and construct it came from, so a future reader can check whether it is
  still true rather than trusting a summary.
- **Findings are dated.** Repositories move. A page describes what was true on
  the date at its top, against the commit named there.
- **No vendored code, no submodules.** Links only. A pinned SHA that silently
  rots is worse than a link that is obviously stale.
