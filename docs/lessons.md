# What I learned

Findings that outlived the code that produced them. Each names the attempt it
came from, so it can be re-checked rather than believed.

## 1. A single state enum encodes half-duplex before you write any logic

`IDLE / LISTENING / THINKING / SPEAKING` in one variable makes "listening while
speaking" unrepresentable — and that is the *only* state a duplex system is in
during a barge-in. voxtral-live's audio path already did the right thing while
the state model denied it was happening.

Two independent variables, input state and output state. Decide this at the start;
retrofitting it means revisiting every transition.

*From: [voxtral-live](attempts/voxtral-live.md), finding 2.*

## 2. Echo cancellation is an architecture decision, not a component you add later

AEC requires one component owning both the played and the captured signal on a
shared timeline. Assistant mark I put capture (Scribe Core) and playback
(Realtime Core) in separate repositories with zero imports between them — a
correct decision under its own value system, and it leaves AEC with nowhere to
live.

The moment the microphone stays open during playback, the decomposition has an
acoustic constraint imposed on it from outside. Module boundaries drawn purely
on logical concerns will get this wrong.

*From: [Assistant mark I](attempts/assistant-mark-i.md), wall 2.*

## 3. A detector that returns a boolean fails on the case it exists for

An echo *suppressor* that drops frames above a correlation threshold discards
genuine barge-ins, because user voice plus echo correlates too. Any
signal-processing step reduced to a yes/no throws away the magnitude that the
hard case depends on.

Keep the residual, not the verdict.

*From: [voxtral-live](attempts/voxtral-live.md), finding 4.*

## 4. Cascade versus native speech-to-speech is the only decision that sets the ceiling

Everything else is execution. voxtral-live executed the application layer better
than Assistant mark I and is still structurally further from full-duplex, because
a cascade needs a complete utterance and therefore needs a turn detector.

Corollary: if you are on a cascade, stop trying to make it duplex and start
making the detector's mistakes cheap. Speculative generation on a stable partial,
cancelled when more speech arrives, buys back the silence timeout for the price
of some wasted tokens.

*From: both attempts.*

## 5. Cancellation needs a monotonic id, and it is cheap to add early

`turnId` / `generationId`, with `begin()` superseding prior work, is maybe a
hundred lines. It is what lets an assistant dispatch a slow tool call, speak an
acknowledgement immediately, and then *silently drop* the result if the user has
moved on. Without it, cancellation is best-effort and late results speak into the
wrong conversation.

voxtral-live's `src/conversation/cancellation.mjs` and `delegation.mjs` are the
reference implementation. They were not hard to write; they were hard to know to
write.

*From: [voxtral-live](attempts/voxtral-live.md).*

## 6. Provider-independence pays for itself in testability before it pays for itself in portability

Assistant mark I's stated reason for `RealtimeSpeechProvider` was swapping models
without surgery. The benefit that arrived first was `fake.ts` — a deterministic
provider implementing the same contract, which makes the entire session lifecycle
testable with no network.

The portability argument is speculative until a second provider actually ships.
The testability argument is collected on day one.

*From: [Assistant mark I](attempts/assistant-mark-i.md).*

## 7. Split the problem the same way twice and you get two halves, not two systems

The clearest thing to come out of reading both repositories on the same day: they
are complementary halves of one architecture. Assistant mark I has the media half
and cannot act; voxtral-live has the application half and cannot stop taking
turns.

Neither is a failure. But the effort spent building each half in isolation is
effort spent not integrating them, and the integration is where full-duplex
actually lives.
