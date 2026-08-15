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

## 4. The model's generation sets the ceiling, and you do not set the generation

Everything else is execution. voxtral-live executed the application layer better
than Assistant mark I and is still structurally further from full-duplex, because
a cascade needs a complete utterance and therefore needs a turn detector.

But the sharper version of this lesson took longer to arrive, because both
attempts assumed the choice was cascade versus native. It is not — it is
[three generations](what-full-duplex-requires.md), and *native speech-to-speech
is only the second*. Gemini Live is native and still ends the turn on silence.
Moving from generation 1 to 2 is a real gain and does not reach duplex; the last
step is a property of model weights, not of architecture.

Two corollaries, one per generation you might be stuck on:

- **On a cascade:** stop trying to make it duplex. Make the detector's mistakes
  cheap instead — speculative generation on a stable partial, cancelled when more
  speech arrives, buys back the silence timeout for some wasted tokens.
- **On generation 2:** the remaining gap is not yours to close. Spend the effort
  on being able to absorb generation 3 without a rewrite, and on the walls that
  *are* internal. See [The model problem](the-model-problem.md).

*From: both attempts, and a correction to this page dated 2026-08-13.*

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

## 7. The capability of a voice assistant comes from delegation, not from the voice model

A full-duplex model has to run inference every frame, so it has to be small, so
it is not very smart. The large labs did not fix this by making it smarter. They
gave it a phone.

OpenAI published the numbers, and the gap is not subtle. On BrowseComp — agentic
web search — Advanced Voice Mode scores **0.7 %** and GPT‑Live‑1 with high
reasoning effort scores **75.2 %**. GPT‑Live‑1 is a small, fast conversational
model. The 75 points are not in it; they are in what it is allowed to call.

Two consequences for anyone building this:

- The application half is not the boring half. It is where measurable capability
  lives, and it is the half you can build without waiting for anyone's weights.
- A generation-3 model obtained without a delegation path underneath is a
  pleasant conversationalist that cannot do anything. That is a worse assistant
  than a generation-2 model that can.

*From: OpenAI's published GPT‑Live evaluations, read 2026-08-13. Mark I now
provides the bounded Tool System half for safe realtime capabilities; cross-
check against [voxtral-live](attempts/voxtral-live.md), which explores richer
delegation and cancellation in a cascade.*

## 8. Different attempts solve different internal walls

The clearest thing to come out of reading these repositories together is that
the projects are complementary without being literal halves. Assistant Mark I
has native speech-to-speech media, provider-neutral contracts, and a bounded
safe Tool System bridge. Mark II adds the missing capable-brain path: immediate
acknowledgement, background reasoning, bounded memory tools, structured
evidence, and same-session delivery. Voxtral-live has cascade-side
asynchronous delegation, cancellation, and echo-suppression work.

None is a failure. The next useful architecture combines those proven
boundaries around a model that can actually sustain generation-3 overlap.
Delegation is not a substitute for that model; it is how a small conversational
model becomes a useful assistant while the model ceiling remains out of reach.
