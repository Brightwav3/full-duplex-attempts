# Assistant mark II — delegated voice intelligence

> **Read as of:** 2026-08-15, against M.A.R.K. II superproject commit
> [`ea026b7`](https://github.com/Brightwav3/Assistant-mark-II/commit/ea026b71df249fd9fdd99c6fe68a2ea1e7a965fe)
> and its `assistant-runtime` pointer at
> [`342dd21`](https://github.com/Brightwav3/assistant-runtime/commit/342dd21e49229bdfe61c909657740b2ffe249fbd).
> **Repository:** [Brightwav3/Assistant-mark-II](https://github.com/Brightwav3/Assistant-mark-II),
> a TypeScript meta-repository with independent core repositories.
> **Attempt:** an advanced half-duplex assistant with a delegated intelligence
> path, not a generation-3 full-duplex model.

## What it was trying

M.A.R.K. II asks a more useful question than M.A.R.K. I:

> Can a realtime voice assistant stay responsive while a more capable reasoning
> system works in the background and returns evidence to the same conversation?

The answer is **yes**. On 2026-08-14, a real Gemini Live session in Czech used a
separately configured `gemini-3.5-flash-lite` model for delegated recall. The
voice model acknowledged the request immediately, the user continued talking,
the delegated model searched and viewed bounded memories through Tool System,
and Gemini Live later spoke the result in the original session. The measured
end-to-end delegation latency was roughly two seconds.

The follow-up run also compared a cheaper delegation model. The detailed
six-call measurement supersedes the earlier rough estimate:

| Metric | `gemini-3.1-flash-lite` | `gemini-3.5-flash-lite` |
|---|---:|---:|
| Average latency | 2.15 s | 0.81 s |
| Median latency | 1.58 s | 0.68 s |
| Cost for six calls | $0.00181 | $0.00276 |
| Completed calls | 6/6 | 6/6 |

From these exact measurements, `gemini-3.1-flash-lite` was approximately 2.65×
slower on average (+165%) and 2.32× slower at the median (+132%), while costing
approximately 34.4% less for the six-call run. These are observed Mark II run
measurements, not a provider SLA or a claim that the cheaper model is always the
better choice. They show that the runtime's provider-neutral delegation
boundary can trade reasoning latency against cost without changing the
voice-side interaction contract.

This is not merely a generic resemblance to an advanced voice assistant. The
architecture is materially analogous to the delegation topology OpenAI publicly
describes for GPT-Live: a realtime voice model on the live media path, a
separate asynchronous delegation path, a deeper model for reasoning and tools,
and results or guidance returned to the ongoing voice interaction. See
[OpenAI's public GPT-Live engineering description](https://openai.com/index/continuous-voice-interaction-with-gpt-live/).

The equivalence is at the delegation architecture boundary, not at the model
generation or infrastructure scale. GPT-Live is described as a generation-3
full-duplex system with GPT-5.5 behind it; Mark II uses generation-2 Gemini Live
with a separately configured text model and a runtime-owned broker. That is
still a real architectural match, and the distinction makes the claim stronger
rather than weaker: Mark II reproduced the delegation shape before having the
same full-duplex model or production transport stack.

### Native versus runtime-managed asynchrony

This is the important distinction. The OpenAI article describes asynchronous
delegation as a capability of the GPT-Live system: the live voice path can
invoke deeper reasoning and tools without blocking its media loop. That public
description does not, by itself, prove that the `GPT-Live-1` model has a
provider-native asynchronous tool primitive exposed directly to every caller.

Gemini Live did not give Mark II that delegation primitive. We therefore had to
build the asynchrony around the voice model ourselves:

```text
Gemini Live asks for intelligence_delegate
        │
        ├─ immediately: accepted + executionId + acknowledge instruction
        │
        └─ asynchronously in the runtime:
             broker → text model → Tool System → Memory Core
             validated result → Delivery Scheduler → same voice session
```

So the precise claim is not "Gemini Live natively supports GPT-Live-style async
tools." The claim is: **Mark II built a GPT-Live-like asynchronous delegation
system around a voice model that did not provide that primitive itself.** The
runtime owns the missing semantics — acceptance, background execution,
correlation, cancellation, deadlines, structured results, and delivery policy.

## Architecture, in one paragraph

The realtime voice model receives `intelligence_delegate` as its only
delegated-capability tool, while retaining the separate conversation-control
tool. The
`assistant-runtime` Delegation Broker mints an `executionId`, owns the execution
lifecycle, and returns an acknowledgement instruction without a result. The
configured text model in Intelligence Core then runs the downstream loop. Its
allowlist contains read-only `memory_search` and `memory_view`; it does not
receive `intelligence_delegate`, so delegation cannot recurse. Tool System
remains the policy, validation, guard, broker, and trace boundary. A validated
`delegation.result.v1` is handed to the Delivery Scheduler, which delivers it
to the same realtime session as `source: "delegation"`, according to
`interrupt`, `when_idle`, or `silent` policy.

### Architecture diagram

The following diagram is the visual overview of the delegated voice path. It is
an explanatory companion to the implementation links below; the code remains
the source of truth for the exact tool catalogue and ownership boundaries.

![Delegated voice intelligence architecture](<assets/Delegation Architecture.png>)

### Delegation timeline

The timeline shows the key user-facing property: delegation starts after the
voice acknowledgement and the user remains free to continue the conversation
while the background execution proceeds.

![Delegation timeline](<assets/Delegation Timeline.png>)

```text
user audio
    │
    ▼
Gemini Live voice model
    │  intelligence_delegate + conversation control
    ▼
Delegation Broker ── accepted + executionId immediately
    │
    ▼
Intelligence Core / separate text model
    │
    ▼
Tool System ── memory_search, memory_view ──► Memory Core
    │
    ▼
validated delegation.result.v1
    │
    ▼
Delivery Scheduler ──► same Gemini Live session
                         source=delegation
```

The important separation is temporal as well as structural:

```text
t0       voice model calls intelligence_delegate
t0+      broker returns accepted, executionId, and acknowledgement instruction
t0..     user can continue talking while the text model works
t1       Tool System mediates bounded memory search and memory view
t2       broker validates delegation.result.v1
t2..     scheduler interrupts, waits for idle, or stays silent
t3       the same voice session speaks the delegated result
```

The result is never inserted as a fake user transcript. It is an explicitly
labelled delegation context event. That distinction is what allows the
conversation to continue without confusing background work with something the
user said.

## The GPT-Live comparison

OpenAI's article describes the same separation of responsibilities that Mark II
now implements:

| Boundary | OpenAI's public GPT-Live description | M.A.R.K. II evidence |
|---|---|---|
| Live voice path | GPT-Live streams audio in and speech out while keeping the media loop uninterrupted | Gemini Live owns the realtime voice session through Realtime Core |
| Delegation path | GPT-Live exposes asynchronous delegation at the system level so deeper reasoning and tools run behind the live path | Gemini Live did not provide that primitive; Mark II built it around `intelligence_delegate` with immediate acceptance and runtime-owned background execution |
| Reasoning model | GPT-5.5 handles deeper search/reasoning while GPT-Live keeps the exchange moving | Intelligence Core runs the separately configured delegation model |
| Tool/application boundary | Application logic, tools, and backend work are separated from the media frontend | Tool System owns policy, validation, guards, tracing, and the bounded memory tool loop |
| Result handoff | The voice model incorporates useful results from the frontier model | `delegation.result.v1` is validated and delivered to the same session as `source: "delegation"` |
| Responsiveness target | Delegation latency is part of the ongoing conversation budget | The hardware run measured roughly two seconds end to end and kept the user conversation available |
| Model ceiling | GPT-Live is described as generation 3 and full duplex; this is separate from the system's delegation orchestration | Mark II remains generation 2; the delegation orchestration is implemented, but the voice model does not claim simultaneous speech or backchanneling |

The diagrams below are included as a visual comparison, not as claims that the
two systems share implementation details. The first makes the topology explicit:
media frontend above, application/delegation layer below, and the deeper model
connected to tools. The second shows the same user-facing delegation idea that
the Mark II hardware run demonstrated.

![OpenAI GPT-Live system architecture](<assets/OpenAI GPT-Live system architecture.png>)

![OpenAI GPT-Live delegation](<assets/OpenAI GPT-Live delegation.png>)

The other two visuals document GPT-Live engineering that Mark II does not claim
to have reproduced: stateful inference handoff/context compaction and the WARP
transport handshake. They are retained here because they define the remaining
distance between a verified application-level delegation system and a
production-scale full-duplex platform.

![OpenAI live context compaction](<assets/OpenAI live context compaction.png>)

![OpenAI WARP handshake](<assets/OpenAI WARP handshake.png>)

## Against the full-duplex criteria

| Criterion | Status |
|---|---|
| 1. Simultaneous input and output | **Partial.** The hardware run proves that the user can keep talking while background delegation runs and that the result can wait behind assistant output. It does not prove generation-3 simultaneous user/assistant speech. |
| 2. Echo cancellation | **Separate Mark II qualification.** The runtime composes an AEC boundary, but delegation itself is not evidence that the assistant can safely hear a user over its own open-speaker output. |
| 3. No blocking turn detector | **No.** Gemini Live remains generation 2 and still imposes provider-side turn-taking. Delegation makes deeper work asynchronous; it does not remove the voice model's turn boundary. |
| 4. Full-duplex model | **No. Generation 2.** Gemini Live is native speech-to-speech but cannot backchannel while the user is still speaking. |
| 5. Interruptibility that reaches the whole system | **Useful partial pass.** Broker cancellation, session-close handling, late-result policy, and delivery modes are explicit and tested. This is whole-task lifecycle control, not proof of generation-3 conversational overlap. |

Mark II therefore does not solve full duplex. It solves a different wall that
matters just as much for a capable assistant: the voice model does not have to
be the system's only brain, and a slow operation no longer blocks the
conversation.

## What was actually verified

### Immediate acceptance without an invented answer

`RuntimeDelegationBroker.accept()` returns `status: "accepted"`, an
`executionId`, and an `assistantInstruction` with `doNotInventResult: true`.
The broker starts observing the execution without awaiting the text model, and
the acceptance payload contains no delegated result. This is covered by the
broker tests and by the composition test that keeps the voice catalogue narrow:

- [`src/delegation/broker.ts`](https://github.com/Brightwav3/assistant-runtime/blob/342dd21e49229bdfe61c909657740b2ffe249fbd/src/delegation/broker.ts)
- [`tests/delegation-broker.test.ts`](https://github.com/Brightwav3/assistant-runtime/blob/342dd21e49229bdfe61c909657740b2ffe249fbd/tests/delegation-broker.test.ts)
- [`src/delegation/intelligence-tool.ts`](https://github.com/Brightwav3/assistant-runtime/blob/342dd21e49229bdfe61c909657740b2ffe249fbd/src/delegation/intelligence-tool.ts)

The runtime does not speak a hardcoded phrase. It gives the voice model a
structured instruction to acknowledge background work naturally, then lets the
voice model choose the wording.

### Separate model roles and a single capability boundary

`createDelegation()` builds the delegated text-model loop independently from the
voice tool registry. The downstream allowlist is exactly the bounded memory
surface, and `intelligence_delegate` is absent from it. The voice side receives
the delegation tool instead of direct memory lookup when delegation is enabled,
alongside the independent conversation-control tool. This keeps authority in
the runtime and Tool System rather than in a model's tool declaration.

- [`src/delegation/composition.ts`](https://github.com/Brightwav3/assistant-runtime/blob/342dd21e49229bdfe61c909657740b2ffe249fbd/src/delegation/composition.ts)
- [`tests/delegation-composition.test.ts`](https://github.com/Brightwav3/assistant-runtime/blob/342dd21e49229bdfe61c909657740b2ffe249fbd/tests/delegation-composition.test.ts)
- [`src/delegation/memory-tools.ts`](https://github.com/Brightwav3/assistant-runtime/blob/342dd21e49229bdfe61c909657740b2ffe249fbd/src/delegation/memory-tools.ts)

### Structured evidence, not a second spoken assistant

The delegated model must produce `delegation.result.v1`. The broker rejects
plain prose, malformed structures, provider failures, and late completions after
cancellation. The result carries memory IDs and provenance instead of pretending
that the background model is the user-facing voice.

- [`src/contracts.ts`](https://github.com/Brightwav3/assistant-runtime/blob/342dd21e49229bdfe61c909657740b2ffe249fbd/src/contracts.ts)
- [`tests/robot-memory-delegation.test.ts`](https://github.com/Brightwav3/assistant-runtime/blob/342dd21e49229bdfe61c909657740b2ffe249fbd/tests/robot-memory-delegation.test.ts)
- [`tests/delegation-text-output.test.ts`](https://github.com/Brightwav3/assistant-runtime/blob/342dd21e49229bdfe61c909657740b2ffe249fbd/tests/delegation-text-output.test.ts)

### Same-session delivery with explicit scheduling

`DelegationDeliveryScheduler` delivers through native context injection when the
provider supports it. `when_idle` queues behind current assistant output;
`interrupt` delivers immediately; `silent` records the result without speaking
it. If native injection is unavailable, the runtime emits an explicit degraded
event rather than silently presenting a different source of truth.

- [`src/delegation/delivery.ts`](https://github.com/Brightwav3/assistant-runtime/blob/342dd21e49229bdfe61c909657740b2ffe249fbd/src/delegation/delivery.ts)
- [`tests/delegation-delivery.test.ts`](https://github.com/Brightwav3/assistant-runtime/blob/342dd21e49229bdfe61c909657740b2ffe249fbd/tests/delegation-delivery.test.ts)
- [`tests/delegation-idle-delivery.test.ts`](https://github.com/Brightwav3/assistant-runtime/blob/342dd21e49229bdfe61c909657740b2ffe249fbd/tests/delegation-idle-delivery.test.ts)

### Hardware result

The manual smoke-test record reports a live Gemini session on 2026-08-14 with
Czech input, immediate acknowledgement, continued user speech, `when_idle`
delivery, native context injection, and a recalled answer matching the stored
memory. It also records the remaining provider-specific unknowns instead of
turning them into successes: Gemini tool calling is deliberately treated as
blocking, native provider result scheduling is not assumed, and the provider's
transcription-language hint was rejected.

- [`docs/delegated-voice-smoke-test.md`](https://github.com/Brightwav3/assistant-runtime/blob/342dd21e49229bdfe61c909657740b2ffe249fbd/docs/delegated-voice-smoke-test.md)
- [`assistant-runtime/README.md`](https://github.com/Brightwav3/Assistant-mark-II/blob/ea026b71df249fd9fdd99c6fe68a2ea1e7a965fe/assistant-runtime/README.md)
- [`assistant-runtime/PROGRESS.md`](https://github.com/Brightwav3/Assistant-mark-II/blob/ea026b71df249fd9fdd99c6fe68a2ea1e7a965fe/assistant-runtime/PROGRESS.md)

## What this attempt proves

Mark II proves that a half-duplex realtime voice assistant can have a separate
intelligence path with the same useful interaction pattern that makes modern
voice delegation feel responsive:

1. acknowledge now;
2. keep the conversation available;
3. reason and use bounded tools in the background;
4. return structured, provenance-bearing evidence;
5. let the active voice model speak it later in the same session.

That is the victory. It moves capability out of the voice model without moving
authority out of the runtime.

It does **not** prove that Gemini Live is full duplex, that the model can
backchannel, that AEC is solved for every device, or that a future generation-3
provider will fit without another integration pass. The architectural bet is
more modest and more useful: when a genuine generation-3 voice model arrives,
the assistant already has a place for the capable brain underneath it.
