# The model problem

> **Surveyed:** 2026-08-13.

[Criterion 4](what-full-duplex-requires.md) says a full-duplex system needs a
full-duplex model, and that no amount of application work substitutes for one.
This page is the practical consequence: what exists, what it costs, and why the
answer is still this thin in August 2026.

It exists because the criteria pages describe a wall without saying how tall it
is. Both attempts in this repository stopped at the same place, and the honest
reason is availability, not effort.

## What is actually obtainable

| Model | Weights | Size | Languages | Hosted |
|---|---|---|---|---|
| [PersonaPlex](https://github.com/NVIDIA/personaplex) (NVIDIA) | MIT code, NVIDIA Open Model License weights | 7B | English | no — self-host |
| [Moshi](https://github.com/kyutai-labs/moshi) (Kyutai) | Apache-2.0 code, CC-BY-4.0 weights | 7B | English | demo |
| Sesame CSM | Apache-2.0 | 1B public | English | demo |
| SALMONN-omni | Apache-2.0 | — | English | no |
| Step-Audio 2 mini (StepFun) | Apache-2.0 | — | Chinese, English | yes |
| GPT‑Live‑1 (OpenAI) | closed | — | many | API announced, not shipped |

PersonaPlex is the strongest open option: built on Moshi, permissively licensed
for commercial use, and explicitly capable of backchannels — the
generation-3 test. It wants **24 GB of VRAM** for full quality at roughly 80 ms
per frame; quantised builds run in 4–7 GB with the fluency that motivated the
exercise degraded. CPU offload is possible and defeats the purpose.

Two constraints apply to every open row:

- **English only.** Moshi states it plainly; PersonaPlex calls multilingual
  support experimental. Multilingual variants are announced but not shipped, and
  when they arrive they will cover large languages first.
- **No hosted API.** These are servers you operate, not keys you obtain.

## Why it is still this thin

Six reasons, roughly in order of how much they explain.

**1. The training data barely exists.** A model learns *when* to speak only from
two-channel recordings with real overlap, interruption, and backchannels — both
speakers on separate tracks. There is no web-scale corpus of that. Moshi
generated much of its own. This is the wall; everything else is a slope.

**2. Duplex forces the model to stay small, and small means less capable.**
Inference runs every frame, continuously, whether or not anyone is speaking, so
the model must fit a real-time budget. Moshi and PersonaPlex are 7B.

The large labs did not solve this by making the duplex model smart. They solved
it with **delegation** — see [lesson 7](lessons.md). A generation-3 model on its
own is a fast conversationalist, not a capable assistant.

**3. Serving economics are hostile.** A turn-based model is idle while the user
speaks; a full-duplex model is never idle. Cost scales with conversation minutes
rather than requests, and the workload is a long-lived stateful session rather
than a request-response call. This also makes a free tier structurally unlikely:
free tiers work for text because requests are short and burstable, and a held-open
GPU session is the opposite of that.

**4. Multilinguality multiplies already-scarce data.** Two-channel conversational
audio, times languages. For small languages the number is approximately zero, and
no commercial incentive changes that soon.

**5. It is hard to evaluate.** "Conversational flow" has no WER. OpenAI had to
build human preference evaluations to measure it. Research optimises what it can
measure, so nobody optimised "does not interrupt awkwardly" until there was a
product reason to.

**6. The feedback loop is asymmetric.** Hosted assistants ship to a hundred
million-plus weekly voice users; open models ship as a repository. That gap is
not about quality, but it produces quality.

## What this means for an attempt

Directly: **stop treating generation 3 as a build target.** It arrives, or it
does not, on someone else's schedule. What an attempt controls is whether it can
absorb one when it appears.

That reframes the other four criteria. Splitting input and output state
([lesson 1](lessons.md)), giving echo cancellation somewhere to live
([lesson 2](lessons.md)), and monotonic cancellation ids
([lesson 5](lessons.md)) are not approximations of full-duplex. They are the
work that decides whether a generation-3 model, once obtained, is expressible in
the architecture around it — or is throttled back to generation 2 by a state enum
and a silence timer.

The reverse also holds, and is the more common failure: a system that adopts a
full-duplex model without doing that work behaves like a turn-based one and
concludes the model was overrated.

## What to watch

- **GPT‑Live API** — announced with a signup form, not shipped. Likely the first
  broadly available generation-3 API, and likely multilingual.
- **Google** — has to answer GPT‑Live. This is the cheapest path for anything
  already speaking to Gemini Live: potentially a model-string change rather than
  a new provider.
- **Kyutai** — hosts its own work and has multilingual Moshi in development; the
  most plausible source of something both open and hosted.
- **Mistral** — already ships Voxtral and a realtime transcription WebSocket; a
  speech-to-speech model would be a short step, and an EU-hosted one at that.

The realistic forecast for early 2027 is generation-3 access **through APIs**,
not through weights you run at home in your own language. Planning around the
API is the better bet; planning around self-hosting is a hardware purchase.
