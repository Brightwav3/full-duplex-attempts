# Glossary

Defined once so the attempt pages can be terse.

**AEC (Acoustic Echo Cancellation)** — removing the assistant's own output from
the microphone signal, using the played audio as a reference. Requires both
signals on one timeline. Distinct from *echo suppression*, which merely mutes or
gates when echo is detected.

**Barge-in** — the user speaking while the assistant is speaking, with the intent
of taking the floor. The case that separates a duplex system from an
interruptible half-duplex one.

**Cascade** — the STT → chat → TTS pipeline. Three models, three hops, and an
intrinsic notion of a turn, because the chat model needs a complete utterance.
Easy to build and to swap parts of; hard ceiling on latency and duplexity.

**Full-duplex** — from telephony: a channel carrying signal in both directions
simultaneously. Here: user and assistant can both speak, and each keeps hearing
the other. Contrast *half-duplex*, which carries one direction at a time and must
switch.

**Generation id** — a monotonic id for one attempt at producing a reply. Late
results carrying a superseded id are discarded rather than delivered. See *turn
id*.

**Backchannel** — "mhm", "yeah", "got it" produced *while the other party is
still speaking*, signalling attention without claiming the floor. The practical
test for a full-duplex model: a turn-based one cannot do it, because emitting
anything means the turn changed hands.

**Delegation** — the voice model handing deeper work (search, reasoning, a tool
call) to a separate, stronger model, and continuing to converse while it runs.
The source of most of a voice assistant's measurable capability; see
[lesson 7](lessons.md).

**Native speech-to-speech** — one model consuming streaming audio and emitting
streaming audio, with no intermediate text. Necessary for full-duplex and not
sufficient: Gemini Live and ChatGPT Advanced Voice Mode are native and still
turn-based. See *generation*.

**Generation (1 / 2 / 3)** — the shorthand this repository uses for the three
architectures: cascade, turn-based native, full-duplex. Defined in
[criterion 4](what-full-duplex-requires.md). The distinction matters because
generation 2 is routinely described as full-duplex, and scoring it that way makes
an unreachable ceiling look like an unfinished integration.

**NLMS (Normalised Least Mean Squares)** — an adaptive filter, the standard
workhorse for software AEC. Learns the room's echo path continuously.

**Partial transcript** — an in-progress transcription that may still change.
Useful for display and for speculative work; dangerous as a trigger for
turn-taking, since every hesitation would fire it.

**Residual energy** — what remains after subtracting the scaled reference signal
at the best-matching lag. Above a threshold, it means the user is speaking. The
magnitude-preserving alternative to a boolean echo verdict.

**Silence timer / turn detector** — "the user has been quiet for N ms, therefore
they are done." A guess that costs its full duration in latency on every turn.
voxtral-live's default is 550 ms.

**Speculative generation** — starting to generate a reply from a stable partial
transcript, and cancelling if more speech arrives. Trades wasted tokens for the
silence timeout.

**Supersede** — the semantics where starting new work invalidates older
in-flight work by id, so results from the abandoned work have no effect.

**Turn id** — a monotonic id for one user utterance and everything derived from
it. The unit cancellation is expressed in.

**VAD (Voice Activity Detection)** — deciding whether a frame contains speech.
Not the enemy of full-duplex; a VAD gating *what gets sent upstream* is fine. A
VAD deciding *when the turn ends* is the problem. Same component, different job.
