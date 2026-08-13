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

Removing the detector entirely requires a model that consumes audio continuously
and decides for itself when to speak — which means criterion 4.

### 4. A model that accepts streaming audio and emits streaming audio

Native speech-to-speech (Gemini Live, GPT Realtime, Moshi) versus a cascade. The
cascade is easier to build, easier to swap parts of, and easier to debug — and it
has a floor on latency and an intrinsic notion of a turn that no amount of
application-layer work removes.

This is the single decision that determines the ceiling. Everything else is
execution.

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
