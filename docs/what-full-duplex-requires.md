# What full-duplex actually requires

The criteria every attempt in this repository is judged against. They exist
because "full-duplex" is used loosely enough to mean anything from "you can
interrupt it" to "there is no notion of a turn at all", and without a fixed
definition the attempts cannot be compared.

Borrowed from telephony: a **half-duplex** channel carries signal in one
direction at a time and must switch; a **full-duplex** channel carries both
directions simultaneously. Applied to a voice assistant, that is not a feature
you add at the end — it is a property of where the audio path is cut.

## The five criteria

### 1. Simultaneous input and output

The microphone stays open, and frames keep being processed, while the assistant
is speaking. This is the minimum. It is also the criterion most often faked: a
system that opens the microphone during playback but discards every frame is
half-duplex with extra steps.

*The tell:* is there a single state enum with `LISTENING` and `SPEAKING` as
mutually exclusive members? If so, the architecture cannot express the state the
system is actually in during a barge-in.

### 2. Echo cancellation

The moment the microphone is live during playback, the assistant hears itself.
Everything downstream — VAD, interruption detection, transcription — is now
receiving its own output mixed with the user's voice.

Three approaches, in ascending order of cost and quality:

- **Gating / detection.** Drop microphone frames that correlate with the
  reference signal. Cheap, and *fundamentally lossy*: when the user speaks over
  the assistant, voice plus echo can still correlate, so the real barge-in gets
  discarded. Exactly the case it needs to survive.
- **Residual subtraction.** Find the best lag, subtract the scaled reference,
  judge the residual energy. A mini-AEC. Strictly better than gating, since it
  degrades to gating in the easy case.
- **True adaptive AEC.** NLMS, WebRTC AEC3, speexdsp, or the OS (Windows Voice
  Capture DSP). Handles nonlinear speaker distortion and drifting clocks.

*The structural requirement:* AEC needs one component that owns **both the played
and the captured signal on one timeline**. If capture and playback live in
different modules with no shared clock, there is nowhere for AEC to live. That is
an architecture constraint, not an implementation detail.

### 3. No blocking turn detector in the audio path

A silence timer — "the user stopped for 550 ms, therefore they are done" — is
a *guess*, and it costs its full duration in latency on every turn, whether or
not it guesses right. Human conversation overlaps; turn-taking is negotiated
continuously, not by silence.

A cascade (STT → chat → TTS) cannot remove the detector, because the chat model
needs a complete utterance to respond to. It can only make the detector's
mistakes cheap: start generating on a stable partial, cancel when more speech
arrives. Wasted tokens buy back the wait.

Note that this criterion asks whether a turn boundary is imposed **anywhere**,
not whether the timer is in your source tree. A provider that ends the turn on
silence server-side has the same failure mode; it has only moved out of view. So
this criterion is not passed by delegating it — it is passed only by criterion 4
being generation 3.

### 4. A model that is itself full-duplex

This is the single criterion that determines the ceiling, and the one most easily
mis-scored, because "native speech-to-speech" and "full-duplex" are routinely
used as synonyms. They are not. There are **three generations**, not two:

| Generation | Shape | Turn detection | Examples |
|---|---|---|---|
| 1. Cascade | STT → chat → TTS | in your code, on silence | voxtral-live, original ChatGPT Voice |
| 2. Turn-based native | one model, audio in and out | provider-side, still on silence | **Gemini Live**, ChatGPT Advanced Voice Mode |
| 3. Full-duplex | one model, continuous both ways | none — the model decides, many times a second | GPT‑Live‑1, Moshi, PersonaPlex |

Generation 2 is a real advance over generation 1: no information is lost to text,
and latency drops. But the turn boundary has been *moved*, not removed. The model
still waits for you to stop before it starts, because it was trained on turns.
Scoring generation 2 as "no turn detector" because the timer is not in your
source tree is the mistake this table exists to prevent — criterion 3 asks
whether a turn boundary is imposed anywhere, not whether you wrote it.

The distinguishing test for generation 3 is the **backchannel**: can the model
say "mhm" *while you are still talking*, without that ending your turn? A
generation-2 model cannot, because emitting anything means the turn changed
hands. There is no application-layer trick that adds this.

A consequence worth stating plainly: **generation 3 is not something you can
build.** It is a property of model weights. You obtain it by renting an API or
downloading weights, and until then every other criterion is work you do to be
ready for it — not work that substitutes for it. See
[The model problem](the-model-problem.md) for what is actually obtainable today,
and at what price.

### 5. Interruptibility that reaches the whole system

When the user barges in, playback must stop, in-flight generation must be
abandoned, and any work already dispatched (a tool call, a search) must be
discarded or reattributed. A system that stops the speaker but still speaks the
previous answer thirty seconds later is not interruptible; it is delayed.

*The tell:* is there a monotonic id — a turn id or generation id — that late
results are checked against before they are allowed to have an effect? Without
one, cancellation is best-effort.

## What is deliberately not on the list

- **Emotional prosody, voice cloning, wake words.** Product features. Orthogonal.
- **Low latency as a goal in itself.** Latency is an outcome of criteria 3 and 4,
  not an independent axis. Optimising it without addressing those is polish on a
  cap.
- **"Removing the VAD."** VAD is not the problem. A VAD that gates *what gets
  sent* is fine and often necessary; a VAD that decides *when the turn ends* is
  the problem. Same component, different job.
