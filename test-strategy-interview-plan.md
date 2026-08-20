# Interview Plan — Senior Test Strategy

**Domain:** Banking — mobile, web, API, batch workloads, event-driven apps, databases
**Format:** Single technical interview, 60 min, one panelist
**Bar:** Senior strategist who challenges ways of working, reimagines testing (incl. AI), and applies standards & techniques to *unblock* work — hands-on depth, not just vocabulary.

---

## 1. Role Context & What the Candidate Must Deliver

- Sets the **testing strategy and standards** for a bank portfolio of 30+ applications.
- **Challenges the status quo** — in particular the "one-size-fits-all" E2E suite that spans the whole bank.
- **Accelerates testing** by applying modern techniques (property-based, mutation, contract testing) and standards to unblock delivery, not slow it down.
- Operates in a **daily-deploy / monthly-release** world driven by **feature toggles**.
- Tests across **workload patterns**: mobile, web, API, batch, event-driven (Kafka), databases.
- **Hands-on knowledge** — can think in concrete techniques, not just talk.

### The reality they must design for
- 30+ interconnected applications in the domain — but the domain is **sandwiched**: upstream core systems feed in, downstream digital channels consume out. The candidate's teams do **not** control the systems on either side.
- **They own the journey/programme** — which means the *responsibility to test end-to-end is theirs by mandate*. E2E cannot simply be "delegated away"; it must be re-architected, not abdicated.
- Journeys criss-cross 30 applications. The E2E suite is "one size fits all," aligned across the bank — and it is the only place the journey is verified today.
- Test environments are **unstable and outside their sphere of influence**.
- Daily deploys to production; monthly customer release; feature toggles everywhere.
- Maturity: manual-heavy to traditional-automation — they are building capability *and* modernising.

**The core tension the candidate must hold:** "we are *answerable* for the end-to-end journey across systems we don't control" **vs.** "a single 30-app E2E suite run on unstable environments cannot be the strategy." The target candidate names this tension unprompted and designs around it — journey-level E2E done *deliberately* (sliced, risk-scoped, prod-canary-backed), component integration verified via contracts at the seams, and residual risk accepted explicitly with the business.

---

## 2. Format & Time Budget

| # | Segment | Time | Signal tested |
|---|---------|------|---------------|
| 1 | Intro / context setting | 5 min | Self-framing, ownership language |
| 2 | **Thread 1 — "Break the model"** | 15 min | Challenge, strategic pragmatism, influence, bank reality |
| 3 | **Thread 2 — "Get your hands dirty"** | 15 min | Technical depth, hands-on knowledge, pragmatism |
| 4 | **Thread 3 — "The contract question"** | 12 min | Integration strategy, CDC vs bidirectional depth |
| 5 | **Swing — AI reimagining testing** | 6 min | Innovation, trust boundary (differentiator, not gate) |
| 6 | Close / candidate questions | 7 min | Curiosity, culture fit |

**Total: 60 min.** If a candidate is weak in one thread, **don't rescue them** — harvest the signal and move on. If strong, extend with the bonus probes below.

---

## 3. Threaded Discussion Model

Each thread opens with a **scenario prompt**, then escalates through a **probe ladder**: from strategy (what would you do) → technique (how exactly) → hands-on (show me you've done it). The ladders let a shallow candidate stay comfortable for 2–3 questions, then collapse under the follow-ups.

The interview is **threaded** — follow the candidate's reasoning, branch into their claims, and push down. Do not read questions off a list.

---

## 4. Scenario Question Banks

### Thread 1 — "Break the model" (~15 min) — centerpiece

**Prompt:**
> "We run daily deploys across 30+ apps with a monthly release train and feature toggles. Our E2E suite is one-size-fits-all — one aligned suite across the whole bank. It's slow, it's brittle, and the test environments are unstable in ways we don't control. Here's the catch: we're a sandwiched domain — upstream core systems feed in, downstream digital channels consume out — and we *own the journey*. The mandate to test the journey end-to-end is ours. You've been hired to challenge this. What's wrong with the model, and how do you re-architect testing so the journey is still verified, but E2E stops being the bottleneck?"

**Probe ladder:**
1. What is *actually* wrong with one-size-fits-all E2E? (correlated failures — the suite fails for reasons unrelated to the change; non-determinism; slow feedback; testing breadth instead of risk)
2. **The tension probe (most important):** "You can't abdicate the journey E2E — it's your mandate. But a single suite across 30 apps on unstable envs doesn't work. Hold both truths: how do you own the journey without a monolithic E2E suite?" (Strong answer: decompose the journey into **slices** — value-path segments with their own seams — each slice verified in CI via contracts/test doubles; a **thin cross-journey orchestration suite** run at a deliberate cadence, not per-deploy; and **production as the ultimate E2E oracle** — canary + telemetry + journey-aligned monitoring. The *remaining* E2E is a consciously scoped, risk-sampled artefact, and the residual risk is an explicit conversation with the business — not an accident.)
3. **Seam probe:** "If teams can't test their slice locally or in CI, is the E2E suite the cause or a symptom?" (Expect: the suite is a *symptom* — teams don't own the seams *within* the journey, so E2E is the only place slices are verified. The fix is test intimacy, not more test tools.)
4. **Sandwich probe:** "You don't control the upstream cores or downstream channels. How do you verify those boundaries without a full E2E run?" (contract testing + service virtualisation/stubs on both sides — the sandwiched domain is precisely where contracts earn their keep; the journey orchestration tests then only need to prove the *flow*, not re-verify each boundary)
5. How do you decide what *must* be E2E vs what must not? (journey slices in CI, boundary contracts, E2E shrinks to orchestration happy-path + production canary/monitoring)
6. **Toggle probe:** "How do feature toggles change what you verify before vs. after release?" (verify per toggle-path in CI, kill-switch behaviour, flag decay, data-path integrity behind a partial rollout)
7. **Env-instability probe:** "The environments are unstable and you can't fix them. Design so testing doesn't depend on them." (contracts + virtualisation move integration verification into CI; ephemeral envs; prod-canary as the last line)
8. **Influence probe (no authority):** "You don't own the teams. How do you land this across the bank?" (evidence from a pilot, make the new way the *path of least resistance*, peer champions, standards as templates not mandates, governance enforced in the pipeline)
9. **Risk probe (bank):** "You've cut the E2E suite by 70%. How do you sleep at night — what gives the bank confidence regression isn't leaking?" (risk-based sampling, mutation analysis of what the remaining suite covers, production telemetry as the oracle, a defined 'we accept X residual risk' conversation with risk/audit)

**Expected observations:**
- **Weak / junior:** Stays inside the model — "we need a better E2E framework, more parallelisation, more testers." Or the opposite failure: **abdicates** — "just push testing down to the component teams," ignoring that the journey mandate is theirs.
- **Solid senior:** Correct textbook — test pyramid/honeymoon, more unit/integration coverage, contract testing named. Holds the journey mandate but can't reconcile it with cutting E2E.
- **Strong (target):** Names the tension unprompted ("I own the journey, but I can't own a 30-app suite") and *designs around it* — journey slices, boundary contracts on both sides of the sandwich, thin orchestration suite at deliberate cadence, production as the E2E oracle, residual-risk accepted explicitly with the business. Challenges the *request* ("why does every change need the whole journey up?"); sequences a 90-day → 12-month roadmap; handles the no-authority reality with evidence + path-of-least-resistance; translates the E2E cut into business/risk language.

**Red flags:** "Make E2E run in parallel / on the cloud" as the *only* answer; blaming the environment team instead of designing around the constraint; **abdicating journey E2E** because "teams own their quality."

---

### Thread 2 — "Get your hands dirty" (~15 min) — technical depth

**Prompt:**
> "A critical payments API is being re-platformed. It also drives a nightly batch reconciliation job and publishes events to Kafka. Today the team only has example-based tests — a handful of scenarios with known inputs and expected outputs. They want to accelerate and raise confidence. You have a day with them. What do you do?"

**Probe ladder:**
1. What's the limitation of example-based testing *here*? (you only test the examples you thought of; the spec is under-tested at the edges; every regression costs a new hand-written example)
2. **Property-based testing:** "What is it, and where does it shine for a payments API?" (invariants the code must satisfy for *arbitrary* inputs — the oracle isn't a fixed answer, it's a law)
3. **Hands-on probe:** "Give me a property for the money-transfer or reconciliation logic." (Strong answers: `balance_out = balance_in + credits − debits` for all transaction sequences; round-trip serialisation; reconciliation totals agree regardless of split/ordering; idempotency — replaying a message yields the same state)
4. **State-machine probe:** "How do you property-test a state machine or event ordering?" (state-machine/sequence models — arbitrary interleavings must preserve invariants; command generation)
5. **Mutation testing:** "What does mutation testing tell you that coverage can't?" (coverage says code ran; mutation says the tests would *catch a real defect*. It tests the tests — escapes are holes in your suite)
6. **Hands-on probe:** "Where do you run mutation testing, and why not everywhere?" (focused on critical pure logic — money math, reconciliation, dedup, ordering — because it's expensive; it's a *quality gate on the test suite*, not a CI routine for everything)
7. **Relating the three:** "Property-based vs mutation vs example-based — how do they compose?" (examples document intent and read well; property tests generate inputs to find counterexamples against invariants; mutation measures whether the suite can *detect* injected defects; reach for each where it earns its cost)
8. **Batch + event probe:** "Apply these to the nightly batch job and the Kafka consumer." (PBT on partitioning/sharding logic and idempotent reruns; mutation on dedup logic; event-ordering invariants; replay idempotency)
9. **Pragmatism probe:** "What do you deliberately *not* do here?" (don't PBT the UI; don't mutation-test everything; don't let the tooling slow the day-job)

**Hands-on depth checks (how to tell talk from done):**
- Can name concrete frameworks for the stack (e.g., fast-check, jqwik, Hypothesis, PropEr, ScalaCheck).
- Knows deterministic seeding, shrinking of counterexamples, and CI cost control.
- Can say when PBT is the *wrong* tool.
- Mutation: knows the concept of surviving/escaped mutants and "coverage is a proxy, mutation is the measure."

**Expected observations:**
- **Weak:** Name-drops "property-based testing" but can't produce one concrete property. Thinks mutation testing = "run it everywhere in CI." Treats techniques as a smorgasbord.
- **Solid senior:** Can explain PBT and mutation correctly, names tools, but leans on examples.
- **Strong (target):** Gives a *real* property for money/reconciliation logic, sequences the three techniques (mutation gate on the critical module, PBT on the pure logic, examples for intent), and applies the same thinking to batch/event patterns. Knows the escalation path — highest-risk logic first, not the whole landscape.

**Red flags:** Cannot give a single concrete property. Says "AI will just write the tests" with no technique underneath.

---

### Thread 3 — "The contract question" (~12 min) — integration strategy

**Prompt:**
> "With 30+ services, daily deploys, and an E2E suite you're shrinking — but you *own the journey*, and the journey crosses upstream core systems and downstream digital channels that you don't control. How do you verify the journey's integrations without paying the full E2E price? Where does contract testing fit — and when would you choose consumer-driven contracts vs bidirectional contract testing?"

**Probe ladder:**
1. What problem does contract testing solve? (integration correctness at the seam, decoupled deploys, catching a contract break in CI instead of in E2E or prod)
2. **Core distinction probe:** "In plain terms, what is the difference between consumer-driven and bidirectional contract testing?" (Strong answer: *consumer-driven* (e.g., Pact) — consumers publish their expectations, the provider verifies it satisfies them; the contract is *driven* by the consumer's needs. *Bidirectional* (e.g., Specmatic, OpenAPI + examples) — a single contract/schema is verified *from both sides*: provider verifies against consumer expectations, consumer verifies against provider's example interactions; often generated from an OpenAPI/schema)
3. **Trade-off probe:** "When do you choose CDC, and when bidirectional?" (CDC: few well-known consumers, consumer team invested, expressive — but coordination-heavy and provider can't see all consumers without a broker. Bidirectional: many consumers, polyglot, low coordination, cheap to adopt from a schema — but shallower, couples everyone to the shared spec's quality)
4. **Sandwich probe:** "You're the meat in the sandwich — upstream cores feed you, downstream channels consume you. Where do the contracts go, and who verifies them?" (Strong answer: contracts at *both* seams — as consumer of the upstream core, as provider to the downstream channel; the sandwiched domain is the boundary where contracts are most valuable because the systems on either side are out of your control — a contract break is caught the moment either side changes, without needing the whole journey up)
5. **Strategy probe:** "How does this interact with your E2E cut and the unstable environments?" (contracts push integration verification into CI — no shared env needed; journey E2E shrinks to orchestration happy-path + prod canary; contracts are the enabler of the daily-deploy model)
6. **Async probe:** "How do you contract-test Kafka / event-driven integrations?" (message/event contracts — consumer expectations on message content, schema compatibility on the topic, versioned evolution)
7. **Bank probe:** "What breaks in a bank context — versioning, breaking changes, many consumers?" (contract broker/registry, change notification, backward compatibility, 'the contract is a deployment gate')
8. **Hands-on probe:** "Your mobile app calls a BFF that calls the payments API, which reads from an upstream core. Where do you draw the contract boundaries?" (contract at the BFF→service seam and BFF→app API seam, and at the upstream-core boundary — not the UI layer; avoid over-contracting — every contract is a coupling)

**Expected observations:**
- **Weak:** "Contract testing = Pact." Can't explain bidirectional. Treats it as a tool, not a strategy. Thinks contracts replace E2E entirely (including journey orchestration).
- **Solid senior:** Explains both correctly, can say when each fits, mentions Pact broker.
- **Strong (target):** Articulates the *strategic* trade-off (coordination cost vs coupling depth), ties contracts to the E2E re-architecture and the env-instability constraint, **places contracts at both sides of the sandwich** as the mechanism that lets the journey be verified without the whole journey being up, handles async contracts, and can draw the seam boundaries for a real topology.

**Red flags:** Can't distinguish CDC from bidirectional beyond tool names. Believes contracts make E2E obsolete instead of shrinking it to its proper role.

---

### Swing — AI reimagining testing (~6 min) — the differentiator (not a gate)

**Prompt:**
> "You've got the same budget, the same 30 apps, the same E2E pain. What's the *single highest-leverage* place to apply AI in our testing landscape — concretely — and what's the honest limit of it?"

**Probe ladder:**
1. **Concreteness probe:** Push past "AI writes tests." (Strong answers tied to *their* pain: AI triage of flaky E2E results; generating property-based *invariants* from specs/ACs that a human validates; anomaly detection on prod telemetry as the regression oracle; mutation-test triage; natural-language ACs → contract examples)
2. **Trust boundary probe:** "Where do you distrust it?" (anything touching money, regulatory, or SLAs stays human-gated; hallucinated assertions that test the *code* instead of the *intent* — the oracle problem)
3. **Human-in-loop probe:** "How do you keep the engineer in charge?" (AI proposes candidates — properties, examples, contracts; the human owns the invariant/intent and the gate)
4. **Future probe:** "How does the test engineer's role change in an agentic world?" (intent-definition, governance, data trust, owning the oracle — not writing the script)

**Expected observations:**
- **Weak:** "Copilot writes my test cases." No trust boundary. No tie to the concrete pain.
- **Differentiator (target):** One concrete, credible use-case aimed at their E2E/env pain, a clear trust boundary, and a view of the engineer as the owner of *intent* — ideally connecting AI-generated properties to the Thread 2 conversation.

---

## 5. Optional Quick Probes (sprinkle if time allows)

- "We test mobile, web, API, batch, events, and databases. Does one testing strategy fit all of them?" (Expect: no — distinct risk profiles, feedback loops, and techniques per workload pattern.)
- "Your nightly batch must finish in a 2-hour window. How do you verify that, and what breaks at scale?" (volume scaling, chunking, restart/resume, idempotent reruns)
- "A consumer lags behind on Kafka in prod. What test should have caught it, and where does it live in the pipeline?" (produce→consume latency, consumer-outage scenarios, lag as a gate)

---

## 6. Scoring Sheet

### 6.1 Anchored scale (applies to every dimension)

| Score | Meaning |
|-------|---------|
| **5** | Exceptional — could credibly own testing strategy for the whole bank; reframes problems, hands-on depth, future-ready. |
| **4** | Strong hire — challenges effectively, sequences change, real technique depth. |
| **3** | Solid senior — correct textbook answers, sound day-job delivery, but stays inside the model. |
| **2** | Shallow — vocabulary without depth; collapses under probe ladders. |
| **1** | Fundamentally unsuitable — gaps in core testing knowledge. |

### 6.2 Weighted dimensions

| Dimension | Weight | Score (1–5) | Weighted |
|-----------|--------|-------------|----------|
| Strategic pragmatism (reframe, sequence, unblock) | 30% | | |
| Technical depth (PBT, mutation, contracts — hands-on) | 30% | | |
| Influence & challenge (no authority, resistance) | 15% | | |
| Bank / risk awareness | 10% | | |
| AI innovation (differentiator) | 15% | | |
| **Overall** | 100% | | |

### 6.3 Select rule

**HIRE** when:
- Weighted overall ≥ **3.5**, **AND**
- No single dimension scores **< 3**.

**Rationale:** ≥3.5 means stronger than a solid senior — correct for a strategist who must *challenge*. The <3 floor prevents someone brilliant on technique but unable to land change (or vice versa) from averaging through. AI is intentionally a differentiator, not a gate — a candidate can score 2 there and still be hired if the core is strong.

### 6.4 Notes / evidence capture
- Record **one concrete evidence quote per dimension** during the interview — verbatim where possible. Scores without evidence are not scores.
- Flag any dimension where the candidate had to be *coached* toward the right answer (reduce by 1 vs. unprompted).
- The two must-pass dimensions are **strategic pragmatism** and **technical depth**. A "no hire" is anyone who fails one of these two.
