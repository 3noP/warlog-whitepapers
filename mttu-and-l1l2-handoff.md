# Mean Time To Understand: Formalizing the SOC Analyst Cognitive Bottleneck and the L1/L2 Escalation Handoff as a Lossy Channel

**Warlog Whitepaper Series**
*April 2026*

---

## Abstract

Security teams usually measure Mean Time To Detect and Mean Time To Respond. Those metrics matter, but they skip one of the most expensive intervals in the loop: the time required to understand what is happening well enough to make a defensible decision. This paper proposes Mean Time To Understand, or MTTU, as a *conceptual lens* for that interval — not as a stopwatch metric to put behind every analyst, which would invite either gaming or surveillance. What is actually measurable, and what this paper argues should be measured, are the operational signals automatically derivable from a passive-capture system: bounce-back rate on escalations, time from L2 receipt to first net-new analytical step, fraction of L2 actions that duplicate L1 actions, percentage of handoffs whose hypothesis-and-evidence slots are empty. MTTU is the why; those signals are the what.

The L1-to-L2 escalation handoff is treated as MTTU's primary amplifier. The core claim is simple: when understanding is transferred through loose prose, the receiving tier reconstructs context that already existed in the sender's head. That waste is architectural, not individual — and crucially, it is *not* solved by asking L1 analysts to fill more forms. The paper assumes throughout that the typed handoff is composed by passive capture from the trace of L1's work, not written by tired analysts at the end of a shift. Programs that try to enforce typed handoffs by discipline have failed for ten years under different names; the architectural alternative is the load-bearing assumption of every claim that follows.

The paper walks through a concrete escalation under both lossy and typed handoff regimes, names the failure modes that traditional dashboards conceal, addresses the cost of implementing high-fidelity handoffs honestly (including the trust-versus-verification distinction that no format can manufacture), and gives SOC managers a small set of questions they can use this week to audit their own handoff quality.

This paper is a companion to *The Learning SOC* foundational paper. Where the foundational paper describes the operating model in which analyst decisions become reusable operational knowledge, this paper drills into one of the most expensive boundaries in that model: the moment one analyst hands an investigation to another.

This paper proposes an architectural lens and a buyer-side framework. Empirical validation of the operational signals it describes (§11) begins through Design Partner deployments in Q3 2026; the specific numerical targets that follow should be read as design intent and observable patterns, not as benchmarked performance data.

## 1. The Metric Gap

Detection and response are easy to timestamp. Understanding is not. That omission distorts how many SOCs think about efficiency.

Between alert creation and response execution, analysts must assemble context, review evidence, test competing hypotheses, and record a decision that others can trust. That interval is often where the real cost of SOC work accumulates, yet it is commonly hidden inside broader service-level metrics.

MTTU gives this hidden interval a name and a structure. The naming matters even though MTTU itself is not a stopwatch metric — calling out the gap is what makes it visible to managers who otherwise see only the transitions a dashboard timestamps. The measurable signals that operationalize the lens are described in §10; they are derivable automatically from a passive-capture system without anyone holding a stopwatch behind an analyst. MTTU focuses attention on the cognitive work that cannot be solved by faster tooling alone; the §10 signals make the consequences of that work observable in production.

### Relationship to Prior Work

Several bodies of work inform this framing.

Operational SOC metrics such as MTTD and MTTR are useful because they timestamp observable transitions. They do not directly measure the analyst's internal work of assembling enough context to make a defensible call. In this paper, MTTU is introduced as a working design metric for that hidden interval rather than presented as an established industry standard.

Naturalistic decision-making and sensemaking research explain why this interval matters. In high-stakes environments, experts do not move from signal to action in one step; they build an account of the situation under time pressure. SOC triage and escalation fit that pattern closely.

Clinical handoff research offers a partial precedent worth naming carefully. Structured handoff programs in medicine — for example, the I-PASS protocol in pediatric care — exist because unstructured transfer forces the receiving team to rebuild missing context, and they have measurably reduced medical errors. The transferable lesson is real: structured artifacts that preserve reasoning beat free-text summaries on receiving-tier outcomes. The disanalogy is also real and worth stating: medical handoffs operate on patient states that are well-bounded and largely classifiable, whereas SOC investigations have an irreducibly creative and exploratory dimension — the senior analyst working a sophisticated intrusion is not following a procedure, they are constructing one. This paper does not argue for SOC handoffs as a clinical-style checklist. It argues for typed *vocabulary* (active hypotheses, ruled-out alternatives, evidence reviewed, open questions, escalation rationale) that travels alongside the analyst's free-form reasoning, not in place of it. The vocabulary scales with case complexity rather than constraining it: a simple case fills few slots; a sophisticated intrusion fills many. The pattern does not break on novel cases because the slots are open vocabulary, not a fixed script.

The information-theoretic language in this paper is intentionally descriptive. "Lossy channel" is used as an analogy for context transfer, not as a quantitative claim about SOC math. The point is operational: if the artifact cannot carry discarded hypotheses, unresolved questions, and reasoning state, the receiving tier will reconstruct them.

Recent autonomy and agentic-SOC materials from major vendors make this even more relevant. Whether the sender is an L1 analyst or an AI triage stage, the receiver still needs a transfer artifact rich enough to continue reasoning rather than start from scratch.

The contribution of this paper is therefore specific: it offers MTTU as a practical lens for SOC design, decomposes the understanding interval into four types of work, and applies the handoff-as-lossy-transfer framing to both analyst and agent-assisted escalations.

---

## 2. A Working Definition of MTTU

At a practical level, MTTU can be thought of as four components:

```
MTTU = context assembly + evidence review + hypothesis evaluation + decision documentation
```

Each component reflects a different kind of effort.

### 2.1 Context assembly

Finding the relevant alerts, events, entities, asset context, prior history, and supporting references.

### 2.2 Evidence review

Reading and interpreting the material that was assembled.

### 2.3 Hypothesis evaluation

Testing competing explanations and deciding which ones remain plausible.

### 2.4 Decision documentation

Capturing the result in a form that can be acted on, audited, or handed to another tier.

This decomposition matters because different components are improved by different architectural choices.

## 3. Why the L1/L2 Handoff Inflates MTTU

In tiered SOC operations, L1 builds an initial understanding state and then escalates selected cases to L2. In many environments, that transfer still happens through a ticket note or a short narrative summary.

That summary is almost always incomplete. It may tell L2 what L1 concluded, but it rarely captures:

- which hypotheses were considered
- which hypotheses were ruled out
- which evidence was reviewed and found unconvincing
- which uncertainties remain open
- why a conclusion was considered tentative rather than final

As a result, L2 behaves rationally by rebuilding part of the context from scratch. The problem is not analyst discipline. The problem is that the handoff channel is lossy.

### 3.1 The Same Escalation, Two Ways

The cost of a lossy handoff is easier to feel through a concrete case than through abstraction. Consider an alert that lands on Tuesday morning: a privileged service principal (`crm-sync`) makes Microsoft Graph API calls with the `Mail.Read` scope from an autonomous system the principal has never used before; concurrently, three users in the finance team receive OAuth consent prompts for a third-party application registered under a redirect domain six days old; a fourth user has just created an inbox forwarding rule to an external address. The L1 analyst spends roughly forty minutes assembling context, ruling out two benign explanations, identifying one open question, and concludes the case warrants L2 attention.

What L2 receives, in most SOCs today, is a ticket that reads approximately:

> *Multi-signal OAuth event involving `crm-sync` service principal and finance team consent grants. New external domain. Possible token theft chain. Escalating for senior investigation.*

Three sentences. L2 opens the case and starts opening tabs: the cloud IdP audit log, the mailbox audit, the consent grant history, threat intelligence on the new domain, the service principal's normal ASN footprint. Within fifteen minutes of receiving the escalation, L2 is rebuilding context that L1 had already produced. Inside thirty minutes, L2 has reached the same intermediate conclusions L1 reached forty minutes earlier. Net new analytical progress: zero. Net time spent: forty-five minutes of L2 work, nearly all of it duplicate of L1.

A typed handoff for the same escalation would carry, alongside the conclusion, the structure of L1's reasoning:

> **Active hypothesis** — credential harvesting chain: third-party app obtains broad mailbox scope via consent grants; attacker uses scope to read mailboxes and create forwarding rules for slow exfiltration. *(Confidence: medium-high. Pattern matches prior case INC-2024-0317.)*
>
> **Ruled out** — legitimate vendor onboarding (Procurement confirms no current onboarding for this app vendor); routine OAuth consent for an internal tool (the redirect domain is external and six days old).
>
> **Evidence reviewed** — `crm-sync` ASN history (12 months, never seen this ASN); consent grant audit (3 grants in 14 minutes, finance team only); mailbox rule audit (1 forwarding rule created at T+90 minutes after first grant); domain WHOIS (registered six days ago, privacy-protected).
>
> **Open question** — does the inbox forwarding rule pre-date or follow the `crm-sync` anomaly? If pre-date, this may be two independent incidents; if follow, single chain.
>
> **Reason for escalation** — multi-signal pattern matches prior consent-grant attack template; senior judgment needed on whether to declare incident now (pulls Legal and DPO in) or extend investigation by 30 minutes for the timeline question above.

L2 reads the bundle in three minutes. The forty minutes of L1 reasoning are inherited, not reconstructed. L2's first analytical move is the open question L1 named — the actual next step. L2's MTTU on this case drops from forty-five minutes to ten.

The difference is not that L2 has become faster. The difference is that L2 is no longer paying for understanding the organization already produced.

### 3.2 Why this is structural, not behavioral

The temptation in most SOCs is to treat the lossy handoff as a discipline problem: train L1s to write better escalation notes, audit them, enforce templates. This rarely works durably. Under volume pressure, free-text templates degrade into the same three-sentence summaries within weeks. The root cause is not analyst willpower; it is that the artifact of escalation is shaped by the channel used to transmit it. Free-text fields in a ticket queue cannot reliably carry typed reasoning. Typed reasoning needs a typed surface.

It is equally important to name what the typed surface *cannot* be: a form L1 fills out. Any program that asks the L1 analyst to compose a structured handoff at the end of each escalation tends to be bypassed under volume pressure within weeks. This is the failure mode that most structured-handoff initiatives of the past decade have run into, and it is a discipline trap, not a tooling problem. The architectural alternative — and the load-bearing assumption of every claim in this paper — is *passive capture*: the platform records the trace of L1's work as the work happens (which option L1 selected, which alternatives L1 dismissed, which evidence L1 opened, which hypothesis L1 pursued and which they ruled out), and the typed handoff is composed automatically from that trace. L1 does not write the handoff; L1 works the case, and the typed handoff is a derived view of what L1 did. The mechanism is described in detail in *The Learning SOC* foundational paper, §4.5. Without this assumption, the rest of this paper does not survive contact with production.

**Honest scope of "passive."** Passive capture is not magical. Roughly speaking, it covers two layers with different mechanisms.

The first layer — about 80% of what flows into the typed handoff — is fully passive: option selections from menus, case ownership transitions, integrations queried against which entities, playbook steps triggered, evidence links opened, alerts dismissed or escalated, and the timing of all of the above. These are byproducts of the analyst working the case in the platform; the analyst does not contribute any extra action to make them captured.

The second layer — the remaining 20% — covers the *semantic* mapping that pure trace observation cannot infer on its own. The fact that L1 ran a query against VirusTotal does not, by itself, tell the system that L1 was testing the C2-communication hypothesis rather than checking sender reputation. For this layer, the platform does the inference work first: it pattern-matches the action sequence against past similar cases, proposes the likely hypothesis structure, and presents it to L1 for one-click confirmation or correction. The analyst's contribution is a click, not an essay. The free-form *"why"* prompt fires only on consequential discards — when L1 declines a high-impact action the system proposed, or when the bundle's confidence on a verdict is genuinely ambiguous and a future analyst would benefit from one sentence of intent. On every other case, no prompt fires.

Anyone who says passive capture is zero-effort and zero-AI-inference is selling magic. Anyone who says it requires L1 to fill forms is selling a product that will be bypassed in six weeks. The honest middle is: trace fully passive, semantic layer inferred and one-click validated, intent free-form only on consequential discards. That middle is implementable and survives production.

This middle has a second consequence worth naming explicitly. It changes the L1's role from *data entry slave* (the form-filling failure) and from *training data for a black box* (the autonomous-SOC failure) into *curator of inferred structure* — the person whose one-click validation or correction is what makes the AI's interpretation operationally trustworthy. The AI proposes, the L1 confirms or corrects, the typed handoff is the joint product. This is a more dignified role than either alternative, and it is the only role under which the L1 has a defensible position against the *"AI is finishing capturing my brain"* fear. §12 returns to this point directly.

**The rubber-stamping risk and how the loop handles it.** A naive implementation of one-click validation can degrade into rubber-stamping: at hour seven of a high-volume shift, the L1 clicks *Validate* without reading, the path of least resistance has won, and the typed handoff for that case is noise that looks like signal. The risk is real and worth naming.

The architecture does not attempt to prevent this through analyst-level monitoring of click speed, accept-rate per individual, or behavioral telemetry on personal reasoning patterns — that path leads to surveillance, not to learning, and it is incompatible with the manager-culture precondition this paper assumes (foundational §9). Instead, the corrective loop already inscribed in the architecture handles fatigue-induced variance at the level where it can be honestly addressed: the team level, not the individual level.

Concretely, three properties of the existing architecture absorb individual capture variance:

- **Capture is continuous and not authoritative on its own** (foundational §4.5). A single tired-hour validation produces an imperfect typed handoff for that case alone — the doctrine layer is insulated from any single capture.
- **Doctrine is earned through convergence and survival, not single signal**. A captured pattern becomes doctrine only when multiple analysts converge on it across cases and when subsequent events do not contradict the captured verdict. A single rubber-stamped FP cannot become institutional doctrine; if it does turn out to be wrong, the contradicting outcome retroactively invalidates the pattern.
- **The L2 inheriting the case is the corrective layer**. Where an L1 capture is wrong, the L2 sees the structure, corrects what is incorrect, and the corrected pattern feeds back into the team's doctrine. The next analyst working a similar case inherits the corrected understanding, not the rubber-stamped one.

Engineering does not eliminate biology. The architecture cannot make every L1 validation equally rigorous, and pretending it can is what produces surveillance instruments dressed as quality controls. What the architecture does is bound the consequence of variance to the case where the variance occurred, and let the loop correct it at the team level over the next similar cases. That is what is honestly available; anything more would either over-claim or surveil.

**Two scope clarifications matter for the validation step itself.**

First, the validation a tired L1 performs at hour four of a shift is *case-level*, not *doctrine-level*. What the L1 confirms with one click is whether the AI's structural mapping matches what they just did on *this* case — affecting only the typed handoff for this case. What becomes *doctrine* (a pattern reused on future cases) does not flow through the L1's validation click at all; doctrine promotion runs on a separate track entirely (foundational paper §4.5): the system aggregates inference patterns across many cases, surfaces them periodically to a senior analyst for review, and only senior-approved patterns become promoted. A tired L1 swiping yes on a misinferred mapping at 5 PM Friday produces, at worst, an imperfect typed handoff for one case. They do not engrave anything in the team's institutional memory by clicking. The bounded blast radius is what makes one-click validation tolerable on case-level inferences; if the same click had doctrine-promotion consequences, the architecture would be indefensible.

Second, validation friction itself should be calibrated to the consequence of the validation, not just to the confidence of the proposal. A routine structural inference on a routine case is correctly one-click — the cost of error is bounded and reversible. An inference that would touch a doctrine candidate (because the case is unusual, or the pattern is appearing for the first time, or the inference would be aggregated into a senior review queue) requires more from the L1 than acquiescence: typing the technique acronym, clicking the specific evidence that supports the mapping, or selecting from explicit options *why* they agree. The friction is small, but it is real, and it scales with what the validation contributes downstream. The Tinder failure mode appears when validation cost is uniformly cheap regardless of consequence; calibrating the cost defangs it.

## 4. The Handoff as a Lossy Channel

Thinking in terms of context transfer is useful. Every escalation carries an understanding state from one analyst tier to another. If the transfer artifact preserves only a fraction of that state, the receiving tier must reconstruct the missing fraction.

This is the essence of context loss. The label borrows from information theory only as analogy — the point is operational, not mathematical. What survives the transfer determines how much of L1's work L2 inherits versus rebuilds. The richer the transfer artifact, the less duplication L2 incurs. The poorer the artifact, the more of L1's investigation L2 silently repeats.

A framing worth making explicit before going further: this paper proposes typed handoffs as an *architectural hypothesis*, not as a guaranteed outcome. The hypothesis is that structured preservation of analyst reasoning — even when the analyst's per-case validation quality varies, as it inevitably does — should reduce the receiving tier's reconstruction cost relative to free-text handoffs, because variable structure exposes its own gaps in a way that free-text prose conceals. The §11 operational signals are how a SOC tests whether the hypothesis pays out on its own data. The paper claims a defensible mechanism by which the bet should pay; it does not claim a guaranteed delta in advance, and any vendor offering a guaranteed delta is overclaiming. The architecture is the bet; the dashboard is the test; both are honest. Neither is sufficient on its own.

The vignette in §3.1 makes the cost visible at the level of a single case. Multiplied across the hundreds of escalations a tiered SOC handles per week, the reconstruction cost compounds into a structural inefficiency that no improvement in detection speed or response automation can offset. A SOC can shrink MTTD to seconds and MTTR to minutes while leaving MTTU untouched, and the team will continue to feel exhausted because the bottleneck has not moved.

## 5. What High-Fidelity Handoffs Must Carry

A high-fidelity handoff should capture more than a conclusion. It should transfer enough structure for the receiving tier to continue reasoning rather than restart reasoning.

At minimum, a structured handoff should preserve:

1. The hypotheses that were considered.
2. The evidence that was reviewed.
3. The hypotheses that remain active.
4. The conclusions reached so far.
5. The open questions that motivated escalation.
6. The hypotheses that were ruled out, with reasons.

The sixth item is usually where prose breaks down. Ruled-out hypotheses are rarely documented in a reusable way, yet they are exactly what prevent the next analyst from redoing work.

### 5.1 The free-form layer is not optional

There is a critique of typed handoffs worth taking seriously: by formalizing reasoning into structured slots, you risk eliminating the space where intuition lives. The senior analyst's *"something feels off about the timing here"* does not fit cleanly into an evidence-and-hypothesis schema, and a SOC that forces every transfer through structured slots alone will gradually castrate the intuitive judgment its best analysts bring.

The architectural answer is that the typed handoff is *not only* structured slots. A high-fidelity handoff carries two layers travelling together with the case:

- a **structured layer** that holds what is mechanically transferable — evidence reviewed, hypotheses active and discarded, open questions, escalation rationale, confidence on the current position
- a **free-form layer** that holds what does not structure cleanly — the analyst's gut feeling about the case, the half-formed pattern recognition that has not yet sharpened into a hypothesis, the *"this looks like INC-2024-0317 but I cannot quite say why"* that an experienced analyst can offer and a junior one cannot

The two layers carry different *kinds* of authority and they are not in competition. The structured layer is authoritative for facts: what evidence was reviewed, what alternatives were ruled out and on what grounds, what questions remain open. The free-form layer is authoritative for judgment: why the analyst sensed that something was off, what the pattern partially reminded them of, what they would want a senior to look at first regardless of what the structured slots say. These are complementary domains of authority, not competing claims for the same authority. The same pattern is familiar from adjacent disciplines — code review tools combine a structured diff with free-form discussion; clinical handoffs combine structured patient state with clinical-judgment narrative — without anyone treating the structured layer as an admission that the practice is unstructurable.

A second point worth naming: the free-form layer in a typed handoff is itself improved over the status quo. Today, intuition that does not structure cleanly migrates to Slack, Teams, or email — channels that are hard to query, deleted on retention policy, and not joined to the case record an auditor can later pull. In a typed handoff, the free-form layer is timestamped, attributable to the analyst, queryable across cases, and bound to the case it informed. Same intuition, dramatically better preservation.

The typed handoff is what makes the structured part *cheap* — composed by passive capture rather than written by hand — precisely so that the analyst's energy can go into the free-form layer where it actually matters. An L2 receiving a high-fidelity handoff should not become a checkbox-validator; the typed slots compress the routine so the free-form layer can carry the senior reasoning.

A handoff that fills only the structured slots and leaves the free-form layer empty is a sign that the analyst was working a routine case, not that the typed format eliminated their intuition. A handoff that fills only the free-form layer is the lossy default this paper exists to address. Both layers, every case.

## 6. Failure Modes Traditional Dashboards Conceal

Lossy handoffs do not announce themselves on a SOC dashboard. They produce a small set of recurring failure modes that look like other things — analyst variability, alert volume spikes, "communication issues" — and that traditional metrics either miss or actively misread. Naming them is the first step to noticing them.

### 6.1 The bounce-back

L2 receives an escalation, opens the case, realizes the handoff is too thin to act on, and returns the case to L1 with "need more context." L1 redoes part of the investigation and re-escalates. Total elapsed time doubles. From the dashboard, this often shows as two separate state transitions; nothing summarizes the full round-trip cost. Bounce-back rate is a uniquely high-signal metric that almost no SOC tracks.

### 6.2 The silent re-investigation

Bouncing a case back is socially expensive — it implies L1 did poor work, which is career-bad in most teams. So L2 absorbs the loss. L2 redoes the work silently, the case advances, dashboards show a fast L1→L2 transition (nine minutes!), and L2's actual time on the case is three times what it should have been. The metric reads "fast escalation"; the reality is "duplicated investigation." This is the most common failure mode and the hardest to see, because everything observable looks healthy.

### 6.3 The reasoning vacuum

The escalation note carries a status change and a one-sentence summary, with no record of what L1 considered, ruled out, or left open. The case state advances; the case understanding does not. Six months later, when an auditor or post-incident reviewer wants to reconstruct why a verdict was reached, the trail goes cold at the escalation boundary. The reasoning lived in the analysts' heads and exited the organization with the next staff turnover.

### 6.4 The senior-as-database

In every tiered SOC, there is one or two senior L1 analysts whose escalations L2 trusts implicitly because long collaboration has built a private decoding key for their shorthand. When those analysts take a two-week vacation, L2 escalation throughput drops measurably. The team's operational continuity is concentrated in two human memories. This is not a celebration of seniority; it is concentrated operational risk.

### 6.5 The drift to chat

Because the ticket cannot carry the reasoning, reasoning moves to Slack, Teams, or email. The case record stays empty. The audit trail lives in chat history that is hard to query, often deleted on a 90-day retention policy, and not reviewable by an auditor under regulatory scrutiny. The organization has externalized its operational memory into ephemeral channels and may not realize it until the day the auditor asks for the chain of reasoning behind a closed case.

### 6.6 What typed handoffs do not solve — trust

There is a failure mode the typed handoff does *not* address, and pretending otherwise weakens every other claim in the paper. Typed handoffs do not manufacture trust between L1 and L2. If the L2 analyst believes the L1 analyst is an underpaid intern racing to close tickets, no schema is going to make L2 stop verifying L1's work — and L2 is right to verify, especially on high-stakes cases.

What changes with typed handoffs is not the *need* for verification but its *cost*. With a lossy free-text handoff, L2's verification is expensive — forty minutes of reconstruction before L2 can confirm or contradict L1's conclusion. With a typed handoff, L2's verification is cheap — three minutes to walk down the evidence chain and confirm the ruled-out alternatives are genuinely ruled out. Cheap verification looks like healthy senior skepticism. Expensive verification looks like the silent re-investigation of §6.2. Same posture, very different cost.

Trust between tiers is earned over time through reviewable track record, not granted by format. A typed handoff is what *enables* the track record to become visible: L1's reasoning is now attributable, queryable, and reviewable across cases. Over months, L2 develops calibrated trust in specific L1 analysts based on visible patterns — *"L1 analyst A is consistently right on phishing-derivative chains, L1 analyst B tends to over-narrow on the first plausible hypothesis"*. This is the trust the format enables, and it is operationally distinct from the trust the format does not manufacture.

The honest framing for a SOC manager is therefore: typed handoffs reduce the cost of verification and create the conditions for calibrated trust to develop, but they do not replace the human work of building that trust through observed performance over time.

### 6.7 The rubber-stamping risk on the other side

If verification becomes too cheap, a different failure mode appears: L2 validates without engaging. A three-minute confirmation walk down the evidence chain is a feature for genuinely routine cases (the well-understood phishing pattern that closes ten times a week), and a danger for the high-stakes case the L2 should have read with active suspicion. Trading silent re-investigation for rubber-stamping is a net loss. The architecture has to address both modes, not pick one.

The defense is calibration to stake. Verification cost should not be uniformly cheap; it should be cheap *only* on cases where the structured layer is genuinely informative and the stakes are routine, and *engaged* on cases where either the stakes are high or the structured layer is thin or unusual. Concretely, the platform differentiates:

- **Routine + clear** — clear-cut FP shape with high pattern match, low business impact, no destructive action proposed: cheap verification is correct, three-minute walk-down acceptable.
- **High-stakes + clear** — clean structured layer but the destination is regulated data, the affected identity is privileged, or the proposed response is destructive: the platform surfaces explicit challenge questions ("the structured layer rules out vendor onboarding because Procurement said no current onboarding — when was that confirmation, and against which vendor list?"), requires the L2 to engage with each question before approval, and may require co-review by a second analyst.
- **Novel or thin** — the structured layer has empty slots, low confidence, or matches no prior case shape: the platform flags the case for engaged verification regardless of stakes, because the structure isn't yet trustworthy.

This is not theoretical. The doctrine governance layer (foundational paper §4.5) tracks which patterns have rubber-stamp risk by correlating verification depth with later outcome quality: if a category of cases gets cheap verification and later turns out to be wrong more often than the team's average, the platform raises the verification bar on that category automatically. Cheap verification is earned by track record, not granted by default.

The framing worth keeping: this is the *anti-Taylorist* design choice the paper makes explicitly. A naive "make verification fast" program reduces every case to the same quick approval — the Taylorist factory model, where every operation is timed and compressed. The Learning SOC inverts that priority: routine is compressed *so that* the exceptional can be sanctuarized. The platform earns the right to be fast on routine cases by being explicitly slow — engaged, challenged, co-reviewed — on the cases where speed would be an attack surface. Going *fast* is not the goal. Going *just* on the routine and *deep* on the exceptional is.

**A critical clarification on what "engaged verification" must not mean.** A poorly designed implementation of this principle would interrupt the analyst with challenge-question pop-ups exactly when the case is most pressing. Friday 17:45, critical alert in the queue, the system pops *"are you sure? consider whether this could be Y instead of X"* — and the analyst, correctly, will despise the tool. This is the trombone-d'Office failure mode: friction injected at the moment of maximum pressure, in the form of an interruption that breaks the analyst's flow when the analyst most needs flow.

Engaged verification, properly designed, is not interruption — and it is not *more information by default* either, which is its own failure mode. Showing a saturated analyst additional evidence by default is throwing them more rope when they are already drowning. The principle is *progressive disclosure with the decision-critical signal sharply foregrounded*, not volume. Concretely:

- A high-stakes case foregrounds the *single most decision-critical element* — *"this action will revoke OAuth tokens for `crm-sync`, which synchronizes the production sales pipeline; expected recovery window is four hours; named business owner is X"* — and keeps the rest one click away. The L2's engagement is provoked by the sharpness of the consequence on screen, not by a wall of additional context. The same progressive-disclosure principle the foundational paper applies to the bundle (§3.0) applies here: the bundle holds everything, the surface shows what matters now.
- For genuinely destructive actions, the friction lives at the *approval gate* — which already exists architecturally via the capability catalog (foundational paper §3.4). The L2 reaches the decision; the platform requires explicit engagement with the consequence at the moment of approval, not as an interruption beforehand. The engagement is bound to the act it gates, not to the deliberation that preceded it.
- For high-impact cases requiring co-review, the second reviewer is added *in parallel*, not as a blocking gate in the L2's workflow. The L2 continues working; the second reviewer engages independently; both must converge before destructive execution. The L2's flow is not interrupted by waiting for the second reviewer's attention.

The principle: friction calibrated to consequence, located at the right moment, taking the form of *one sharply-presented decision-critical signal* rather than either interruption or volume. A system that asks a tired analyst "are you sure?" at 17:45 has misunderstood the architecture. A system that pre-loads them with additional evidence by default at the same moment has misunderstood it differently, in the same direction. A system that, at 17:45, presents the single sentence *"this kills production sync for four hours; sign-off required from owner X"* has located the friction correctly.

**Three constraints on what foregrounding may not do.** Progressive disclosure with AI-curated foregrounding creates its own attack surface — if the AI hides ninety percent of evidence to "help the analyst focus" and the rare attack signal is in the hidden ninety percent, the system has produced selective blinding. Three constraints prevent the failure.

- **The L2 can always expand the surface.** Foregrounding is a default, not a constraint. One click reveals the deprioritized evidence. The system does not lock the analyst into the AI's view of what matters; it offers a default and supports immediate override.
- **The filtering criteria are explicit to the analyst.** When the system foregrounds *X* and deprioritizes *Y*, it says so: *"showing this because most decision-critical for technique T; click to see what was deprioritized and why."* The analyst sees not only the foreground but also the rationale for the foreground; they can evaluate whether that rationale fits this case.
- **For high-stakes cases, foregrounding is more conservative, not less.** The cost of hiding is higher than the cost of overload when the stakes are high. The system shows more by default on cases that touch regulated data, privileged identities, or destructive actions; the progressive disclosure compresses for the routine case and expands for the consequential one. The filtering itself is calibrated to stake, the same principle that governs verification depth.

The filtering logic that decides foregrounding is itself doctrine-governed (foundational paper §4.5): the system tracks cases where deprioritized evidence later turned out to be decision-critical and feeds those outcomes back into the filtering criteria. AI-curated foregrounding is not an opaque oracle; it is a doctrine layer that earns its trust the same way every other doctrine layer does, and it can be challenged the same way.

**A long-term concern worth naming honestly: skill atrophy.** Over time, analysts who only ever see foregrounded views may lose the muscle for raw-evidence investigation — the day the AI's foregrounding misses the rare attack signal, an analyst trained exclusively on the curated surface may not know where to look. This is the same concern aviation faces with autopilot dependency, that radiology faces with AI-assisted reading, and that any cognitive-augmentation tool faces over a multi-year horizon. It is not a hypothetical risk and the paper would be incomplete to ignore it. Three practices keep the risk bounded: a fraction of cases is routinely worked in *raw mode* with no foregrounding (calibrated to maintain skill without overwhelming routine throughput); investigation training programs explicitly include unfiltered-evidence drills so junior analysts develop the underlying competency before relying on the curated surface; and the *expand all* affordance must remain prominent and habitually used by senior analysts, modeling the behavior for the team. Atrophy is not eliminated — no augmentation tool is free of it — but a SOC that builds these practices into its operating cadence retains the capability the curated surface depends on for its own correction. A SOC that does not is making a one-way bet on its tool's filtering accuracy. That bet is unsafe over a long enough horizon, and the paper recommends against it.

A SOC manager who watches for these seven modes — rather than for MTTR alone — sees a different picture of operational health than the dashboard suggests.

---

## 7. Adversarial Considerations: Why Format Is Not a Substitute for Suspicion

Clinical handoff research and operational excellence literature share a property that the SOC does not: they assume the patient is not actively trying to deceive the team handing them off. SOC investigations operate against an adversary who studies the defender's tooling, who can craft signals to look like the defender's typical false-positive shape, and who exploits exactly the patterns of trust the defender has built. Any paper on SOC handoffs that ignores this is incomplete by definition.

This is also where the SOC departs from the office-assistant framing that most current AI-in-security pitches inherit. An office assistant operates in a cooperative environment; an SOC operates in a contested one. The AI in a Learning SOC is not productivity software — it is a component of a *weapons system* deployed against an active adversary, and it has to be designed under that assumption from the first line of architecture. Acknowledging that an attacker can attempt to pollute the team's doctrine, exploit cheap verification, or craft signals that fill the structured slots in misleading ways is not a confession of weakness; it is the baseline of seriousness. A paper or product that does not address it is not yet operating in the same problem space.

Three adversarial scenarios are worth naming explicitly, because they constrain what typed handoffs can and cannot do.

**Scenario A — the well-explained false positive.** An attacker who understands the structured slot vocabulary can craft an attack chain whose surface signals look like a clean false positive: plausible benign explanation, low confidence on the attack hypothesis, ruled-out alternatives that read clean. The L2 who relies on the structured layer alone — *"the slots are filled, the confidence is low on attack, L1 ruled it out for a clean reason"* — is precisely the L2 the attacker is targeting. The defense is that structured signals must never override evidence: the verification step still walks the source data, not just L1's interpretation of it. The structured layer compresses what L1 already checked; it does not replace L2's responsibility to check the things L1 might have missed.

**Scenario B — exploiting trust earned by track record.** Once L2 has developed calibrated trust in a specific L1 (§6.6), an attacker who can profile L1's typical FP shape can craft an attack designed to land in exactly that shape. *"L1 analyst A is consistently right on phishing-derivative chains — and now here is a phishing-derivative chain that is actually a credential-harvester for a regulated-data endpoint."* The defense lives in the doctrine governance layer (foundational paper §4.5): patterns that earn cheap verification are revisited periodically, and any pattern whose past instances later turn out to have included an attack chain is demoted — all similar future cases re-flagged for engaged verification, regardless of which analyst owns them. Trust earned by track record is not permanent trust; it is conditionally renewed against later evidence.

**Scenario C — drowning the intuition.** An attacker can produce a chain whose structured layer is uniformly clean — every slot fills cleanly, every alternative rules out cleanly — but where the timing, the ASN sequence, or some other gestalt feature would set off a senior analyst's intuition if they were forced to look at it. A SOC that propagates only the structured layer has just lost its best adversarial signal. The free-form intuition layer (§5.1) is not a nice-to-have for these cases; it is the architectural primitive that prevents a clean-looking structured layer from drowning the senior reasoning that catches sophisticated intrusions. *"Evidence reads clean but the timing feels off"* must survive the propagation, not be implicitly subordinated to the structured layer's cleanliness.

The deeper principle: typed handoffs reduce noise so the senior analyst's adversarial suspicion can hear signal more clearly. They do not substitute for that suspicion. A SOC that treats typed handoffs as a way to *replace* senior judgment with formatted L1 output is building exactly the operating model an adversary wants to face. The right model uses typed handoffs to make routine investigation cheap, so senior judgment has the bandwidth to engage where engagement actually matters — and the doctrine governance layer ensures that "what counts as routine" is itself revisable when adversarial reality contradicts the team's prior calibration.

A buyer who reads this paper should understand that none of this is automatic. Typed handoffs are a *condition* for adversarial robustness, not a *substitute*. The substitute is engaged senior analysts working a SOC where the routine has been compressed enough that they have the bandwidth to engage. That is the operating model the Learning SOC describes; this paper formalizes one boundary of it.

**The doctrine layer is a trade, not a victory.** Adding institutional memory to a SOC creates a new attack surface — adversarial doctrine pollution — that a SOC without institutional memory does not have. This is real, and a paper that did not name it would be incomplete. The honest framing is that the doctrine layer trades one set of attack surfaces for another: it reduces some (concentrated reliance on a few senior analysts whose departure erases operational memory; slow recognition of recurring patterns; investigations that restart from zero on each similar case) and creates others (pollution of the doctrine through crafted "well-explained false positives," exploitation of accumulated trust in specific analyst patterns). The bet this paper makes is that the net effect is positive in environments where the reduction outweighs the new exposure — typically environments with mature threat-intel feeds, senior analyst capacity to engage with §7.1 and §7.2 governance, and acceptance that no architectural choice eliminates adversarial exposure entirely. The bet is defensible. It is not a guarantee. A SOC manager evaluating this paper should weigh which set of attack surfaces their environment cares more about, rather than treating the doctrine layer as pure upside.

### 7.1 What the system does to defend its own doctrine

It is not enough to say *"trust the senior analyst's intuition as the catch."* That answer is real, but treated alone it is also a deflection — the system would be quietly transferring the cost of its own adversarial vulnerability onto the human safety net. A SOC platform that takes adversarial robustness seriously has to do active work on its own doctrine, not just hope its users will catch what it missed.

Four system-side defenses are required, alongside the human one.

- **Pattern-anomaly detection on the doctrine itself.** The system watches not only individual cases but also the *rate and shape* at which doctrine entries are being created. A sudden cluster of "well-explained false positive" entries in a previously rare technique area is itself a signal — flag for senior review independently of whether any individual case looks suspicious. The adversary attacking the doctrine attacks at scale; the system can detect at scale.
- **Adversarial validation of recent doctrine.** Periodically (weekly is reasonable; monthly is the floor), the system generates synthetic adversarial test cases against recently-promoted doctrine entries and runs them against the same dismiss-or-escalate logic. Any doctrine entry that would *accept* an adversarial case as routine is demoted and queued for senior re-review. This is fuzzing for SOC doctrine — the same idea as fuzz-testing security software, applied to the rules the SOC operates by.
- **Cross-correlation with external threat intelligence.** When a CTI feed publishes a new attack technique or a new IOC family, the system automatically scans recent dismissals and suppressions for matches and re-flags any cases that were closed under doctrine that did not yet account for the new technique. The retroactive re-flag is what catches the slow-burn attack that was dismissed two months ago because the doctrine didn't yet know to look for it.
- **Time-decay and re-validation of doctrine.** Doctrine entries are not permanent assertions of truth; they are time-bound observations about a threat landscape that moves. Each entry carries a re-validation interval (technique-appropriate: short for fast-moving categories like commodity malware, long for stable categories like authentication primitives). When the interval expires, the doctrine is re-presented for senior review against current threat intelligence; entries that no longer match current reality are demoted or revised.

These four defenses are not a substitute for the senior's intuition, and the senior's intuition is not a substitute for them. Both are needed. A platform that names only the human defense is shifting blame to its users; a platform that names only the system defenses is over-claiming its own robustness. The honest position is that adversarial robustness is a shared responsibility between the system's active doctrine self-checking and the human's irreplaceable suspicion — and the operating model is designed for both to do their part.

### 7.2 Two preventive defenses, and the limits of prevention

The four defenses in §7.1 are largely retroactive. Pattern-anomaly detection sees clusters after they form; adversarial validation runs periodically after promotion; CTI cross-correlation triggers after a new technique is published; time-decay re-validates after the interval expires. A platform that names only retroactive defenses is, fairly, watching the burglary tape after the burglar is gone.

Two preventive mechanisms are worth inscribing alongside, with care to keep them within bounds where they actually help.

- **Asymmetric promotion thresholds on high-risk doctrine shapes.** Some doctrine shapes are structurally more exploitable than others. Doctrine that says *"this looks like a routine FP because the source is a new vendor"* is more dangerous than doctrine that says *"this is FP because the user explicitly requested the action and the request matches a logged change-management ticket."* The system encodes priors on which shapes have higher exploitation potential and raises the promotion bar on those shapes by default — more cases required for convergence, longer survival window, mandatory co-review by a second senior. Risky shapes earn trust slowly; routine shapes earn trust at the normal rate. This is prior-knowledge encoding, not user-facing instrumentation, and it operates in the doctrine governance layer without affecting the analyst's daily work.
- **Rate limits on sensitive technique areas.** Doctrine in technique categories with high adversarial exposure — credential access, OAuth abuse, supply chain compromise, identity provider attacks — requires longer cooldown periods between candidate identification and promotion, mandatory senior review (not just senior approval), and a higher convergence threshold than other categories. Adversaries invest most in crafting attacks that look routine in exactly these categories; the doctrine governing them should be hardest to fill quickly.

These two preventive mechanisms operate in the background of doctrine governance. They do not affect the analyst's queue, do not require the analyst to do anything differently, and do not introduce instrumentation the analyst could reasonably resent.

The honest limit worth naming explicitly: prevention in this domain is partial, and over-engineered prevention is its own failure mode. A platform that injects synthetic adversarial signals into the analyst's live queue to *"test team robustness"* is gaslighting its operational users — the day the analyst learns they spent an hour on a synthetic test, the trust the platform has accumulated collapses. Live-queue red-teaming of this kind is not on the table, and any platform proposing it has misunderstood what its operational users will tolerate. Adversarial robustness in production is achieved by the combination of the four retroactive defenses (§7.1), the two preventive thresholds above, and the senior analyst's irreplaceable suspicion (§5.1, §6.6). Anything beyond that boundary moves out of architecture and into theater.

---

## 8. Architectural Consequences

Once MTTU is treated as a design problem, several implications follow.

### 8.1 Context assembly should be staged

The next tier should inherit an organized context package, not a raw pointer back to the original alert.

### 8.2 Handoffs should be typed

The transfer artifact should expose stable slots for evidence, active hypotheses, discarded hypotheses, uncertainty, and next actions.

### 8.3 Escalation criteria should be explicit

Escalation should not depend only on analyst instinct. The system should make clear why a case moved to the next tier.

### 8.4 Documentation should be operational, not narrative-only

Free text remains useful, but it should sit alongside structured state, not replace it.

### 8.5 The capture must be passive

This is where most "structured handoff" programs die. If L1 has to fill out a typed form at the end of each escalation, the form will be bypassed under volume pressure within weeks. The discipline-based approach has been tried and consistently fails. The architectural alternative is *passive capture*: the platform records the trace of L1's work as it happens — selections, dismissals, evidence opened, hypotheses pursued, options considered and rejected — and the typed handoff is composed automatically from that trace. L1 does not write a structured handoff; L1 works the case, and the structured handoff is a typed view of what L1 did. This pattern is described in detail in *The Learning SOC* foundational paper, §4.5. It is the load-bearing assumption of every claim in this paper about reducing MTTU at scale.

## 9. What This Costs to Implement

Honest about cost. A structured handoff program is not free, and three failure modes follow from underestimating the cost.

**The form-filling trap.** If the typed handoff is implemented as a form L1 fills at the end of each escalation, the form tends to be bypassed within weeks under any real volume pressure. Analysts who are timed on closure speed cannot easily afford to write essays. The more durable answer is the passive-capture pattern named in §7.5: the platform composes the typed handoff from the trace of work, not from a separate documentation step. Programs that skip this and rely on analyst discipline have, in published practitioner accounts and in our own experience, tended to roll back to free-text within a few quarters. This has been observed repeatedly enough to call it a pattern rather than a coincidence.

**The doctrine-authoring cost.** The escalation template — what fields a typed handoff carries, what counts as "evidence reviewed", what level of confidence triggers required uncertainty fields — has to be authored by the team itself, from existing practice. No vendor template will fit a specific SOC's investigation patterns out of the box. This is two to four weeks of work for a small senior team to define the categories, validate them against recent escalation samples, and tune. The cost is real but bounded; once the doctrine exists, it stops growing.

**The cultural cost of changing what "escalation" means.** In many SOCs, escalation is implicitly a request for help — "I'm stuck, please help" — rather than a transfer of analytical state. Moving to typed handoffs means escalation becomes "here is where the case stands; here is what I'm asking you to decide." This is a different stance and some L1 analysts initially feel it as exposure ("now my reasoning is visible"). Management support to frame typed handoffs as professional work product, not as performance review, is the difference between adoption and silent resistance in the first quarter.

A team that tries to introduce typed handoffs everywhere at once usually ships nothing. The realistic adoption pattern is to start at the highest-cost transition — usually L1→L2 on technical investigations where the rebuild cost is largest — make that boundary typed and passively captured, validate the gain over a quarter, and extend from there. The compounding return appears around month three or four; the team starts inheriting better priors on similar cases, the bounce-back rate drops measurably, and the senior-as-database risk begins to dissolve.

**One precondition that has to be named, because no architecture survives without it.** This model assumes that management will treat the visibility it creates as developmental rather than punitive — coaching grounded in specific cases, attribution of doctrine to its authors, contribution recognized as the team's collective product. A management culture that weaponizes contribution data into per-analyst performance comparison or punitive review is not implementing this model; it is implementing a different operating model under the same name, and the typed-handoff and passive-capture layers will produce stress, resistance, and eventual rejection regardless of how cleanly they are built. *Organizations whose management culture is structurally punitive should not adopt this paper's recommendations.* This is a precondition with the same weight as the Git literacy and CMDB-equivalent preconditions named in *The Learning SOC* foundational paper §9.

A predictable objection: *"saying 'don't buy if your culture is punitive' is the architect's defeat — you're admitting the technology can't protect users from their own managers."* The objection is partly fair and worth answering directly. Every visibility-enabling technology amplifies the culture of the organization that wields it. A spreadsheet can support intelligent budgeting or surveil employee bathroom breaks. NIST CSF works for organizations that want governance and fails for organizations that want compliance theater. The Learning SOC works for organizations that want operational discipline and fails — sometimes badly — for organizations that want a more precise instrument for punishing analysts. Pretending otherwise is what platform vendors do when their pitch needs to fit any buyer; refusing to pretend is what serious operating-model papers do. The honest sale is *"if you are a developmental team, this multiplies your capacity; if you are a punitive team, it multiplies your toxicity; choose accordingly"* — not *"this neutralizes management dysfunction through architecture."* The architecture constrains where it can constrain (§11.4 dashboard guardrails, §3.2 case-level blast radius, §7.2 doctrine governance bounds). The rest depends on culture, and the paper says so.

The alternative — designing an opaque tool to "protect" analysts from their managers — does not actually protect them. Managers determined to surveil find other instruments; opacity hides the surveillance without preventing it. Choosing visibility honestly, with named preconditions on the culture that wields it, is the more defensible position than choosing opacity dishonestly.

## 10. What MTTU Does Not Mean

MTTU is not a claim that all analytical work can be standardized away. The hard part of investigation still involves judgment, creativity, and experience. Structured handoffs do not eliminate expert reasoning. They reduce avoidable reconstruction so that expert reasoning can start from a better prior.

MTTU is also not presented here as an industry-standard KPI. It is a conceptual lens that any SOC can use to examine where analyst time is actually being spent. Treating it as a hard target invites two specific abuses: gaming (analysts close cases faster by skipping reasoning capture) and surveillance (managers track per-analyst MTTU as a productivity metric, which destroys the willingness to admit uncertainty in handoffs). The right use of MTTU is comparative across workflows, case types, and architectural changes — not as an individual performance metric.

## 11. The Operational Signals That Are Actually Measurable

MTTU itself is not directly observable. Treating it as a stopwatch metric is a category mistake — analysts take breaks, parallel-work multiple cases, get pulled into meetings, and any clock-based MTTU number conflates real understanding work with operational interruption. This is precisely why §10 warns against MTTU as a hard target.

What *is* measurable, automatically and without surveillance, are the operational signals a passive-capture system produces by observing the trace of analyst work on each case. These signals are not MTTU; they are the operational evidence for what MTTU diagnoses, and they are concrete enough to put on a dashboard a CFO can read.

### 11.1 The Five Signals

- **Cognitive Loss Coefficient** — the fraction of L2 actions on a case that repeat actions L1 already performed (same query against the same source, same evidence opened, same enrichment requested). Computable automatically from the action log; the platform compares L2's action sequence against L1's and counts the overlap. *This is the headline signal a SOC manager should watch.* A coefficient at 60% means L2 is silently redoing 60% of L1's work — which means roughly 40% of the SOC's senior analyst budget is being thrown away every month to reconstruction that never produced a single new analytical step. Multiplied across hundreds of escalations per quarter, this is the line item the CFO has been failing to see for a decade. The intervention target is below 15%.

- **Bounce-back rate** — fraction of escalations the receiving tier returns to the sender within N hours, measured automatically when case ownership transitions reverse. High bounce-back rate = lossy handoffs that L2 is openly rejecting. Most teams find their bounce-back rate is low not because handoffs are good, but because bouncing back is socially expensive (§6.2). This metric is most informative read alongside the Cognitive Loss Coefficient: low bounce-back + high coefficient = the silent re-investigation regime; both low = healthy handoff regime.

- **Time-to-First-Net-New-Analytical-Step** — the system timestamps every analyst action against the case; *"net-new"* means an action L1 had not already performed. The gap between dashboard transition time (L2 acknowledged in 9 minutes) and net-new-step time (L2 did something L1 had not done in 47 minutes) is the silent re-investigation cost from §6.2 in clock terms. The intervention target is to compress the gap below five minutes on routine cases.

- **Empty-slot rate on handoffs** — percentage of handoffs whose hypothesis-and-evidence slots are filled vs empty. An escalation with no ruled-out hypotheses, no open questions, and no evidence references is a reasoning vacuum (§6.3). The intervention target is above 80% of handoffs with all required slots populated.

- **Reasoning-attribution depth at closure** — for closed cases, how much of the final verdict's reasoning is reconstructable from the case record alone (vs requiring analyst re-interview). Approximated by counting structured slots referenced in the closure rationale and free-form intuition entries timestamped during the case. The intervention target is 100% of closed cases verdict-reconstructable from the record alone.

### 11.2 What a Healthy Dashboard Looks Like

The five signals together form a picture the CISO and the CFO can act on. A healthy SOC, after eighteen months of typed-handoff and passive-capture adoption, should read approximately:

| Signal | Healthy range | Unhealthy range |
|---|---|---|
| Cognitive Loss Coefficient | < 15% | > 40% |
| Bounce-back rate | < 5% (combined with low coefficient) | any value combined with high coefficient |
| Time-to-First-Net-New-Step (median, routine cases) | < 5 min | > 20 min |
| Empty-slot rate | < 20% | > 50% |
| Reasoning-attribution depth at closure | > 90% | < 60% |

What no single number captures, but the five together do: whether the SOC is shipping understanding alongside tickets, or only tickets.

### 11.3 Reading the Signals Against Architectural Change

The signals are most useful when read across an architectural intervention rather than as static targets. A SOC that introduces typed handoffs and passive capture should expect the Cognitive Loss Coefficient to drop within one quarter as L2 stops reconstructing L1's work; the Time-to-First-Net-New-Step to compress within the same window; the Empty-slot rate to drop steeply once passive capture composes the structured layer automatically; and the Reasoning-attribution depth to climb as the doctrine governance layer accumulates. If the signals do not move after an intervention, the intervention has not landed in production — the data tells the team where the program is real and where it is theater.

None of these requires a stopwatch behind any analyst. All of them are byproducts of a passive-capture system. None of them is gameable in the way per-analyst clock metrics are gameable. They are the *measurable* expression of the *conceptual* MTTU lens, and together they let a SOC manager show a CFO exactly where the budget is being spent on reconstruction versus on net new analytical work.

### 11.4 Ethical Guardrails on Dashboard Use

A dashboard concrete enough to put on a CFO's desk is also concrete enough to misuse as a per-analyst whip. The cognitive loss coefficient at 60% is a useful operational signal at the team level; the same coefficient broken out per L1 analyst is the foundation of a micro-management apparatus the Learning SOC explicitly rejects. The guardrail cannot be a policy hope; it has to be an architectural constraint.

Three constraints are inscribed in the design.

- **The five signals render at team, workflow, or case-category level only.** The platform structurally does not surface per-analyst breakdowns of the cognitive loss coefficient, bounce-back rate, time-to-first-net-new-step, empty-slot rate, or reasoning-attribution depth. This is not a UI default that a manager can flip; it is a hard constraint in the data layer. Per-analyst slicing of these metrics is not a feature.
- **What *is* surfaced per analyst is contribution-positive, not performance-comparative.** The dashboard shows: how many doctrine entries an analyst authored that were later reused, which suppression rules carry their attribution, how their submitted reasoning contributed to closed cases, how their flagged intuitions connected to later confirmations. The frame is "what did this person contribute to the team's institutional memory" — not "how does this person's efficiency compare to their peers." This is the architectural expression of the *analyst as author* framing in §12.
- **The architecture does not collect per-analyst engagement telemetry.** No platform-side tracking of individual click speed, individual accept-vs-correct rate, or individual reasoning-quality scores. The corrective loop described in §3.2 operates at the team level — convergence across analysts, doctrine governance, L2 correction of L1 capture — not by surveilling the analyst whose individual capture happened to fluctuate on a tired Friday. A manager who wants comparative per-analyst performance data must use a different surface entirely; this paper neither endorses nor enables that surface.

These constraints are not philosophical preferences. They are the difference between a system that *measures the cost of reconstruction* (the Learning SOC's intent) and a system that *measures the productivity of individual analysts* (a surveillance tool with a different purpose, different ethics, and different failure modes). A platform that allows the dashboard to slide from the first into the second is, in practice, a different product. The architecture should refuse that slide.

### 11.5 What This Dashboard Is, and What It Is Not

The five signals in §11.1 are descriptive. They tell a SOC manager where the team is, where the cost is concentrated, and how those quantities move when architectural interventions land. They do not predict, with calibrated confidence, what intervention X will produce in delta Y on this team in this environment.

This paper makes no predictive claims, and the dashboard makes none either. Anyone selling *"deploy our product and your MTTU drops by N%"* is selling a number they cannot defend — predictive models in this domain would require cross-tenant datasets, stable threat landscapes, and clean confound separation that do not exist in any vendor's hands today. A platform claiming the number is either extrapolating from a single deployment or fabricating it outright.

The honest position is simple. This is a thermometer. It tells you where the SOC is bleeding and lets you watch the bleeding stop, slow, or worsen as you change things. It does not tell you in advance how much a change will help. That is a meaningful capability — most SOCs cannot answer *"where is our reconstruction cost concentrated"* today, and the dashboard makes that question answerable for the first time. It is also a bounded capability, and the paper claims nothing beyond it.

If predictive capability becomes possible later, on the basis of data that does not yet exist and governance that has not yet been built, that will be the subject of a different paper. This one earns its credibility by refusing to promise it.

## 12. What This Means for the L1 Analyst

There is a question this paper will be asked the moment it lands in the inboxes of the people whose work it most depends on: *"My L1 is paid the minimum the market allows, and they are afraid that AI will replace them as soon as it has finished capturing their brain. Why would they ever cooperate with this?"*

The question is honest and the fear is warranted. It deserves a direct answer rather than a comforting deflection.

**The L1 analyst is right to fear the wrong AI.** An autonomous-SOC platform that absorbs L1 reasoning into an opaque model — that suppresses alerts, classifies cases, and closes tickets without preserving why — is genuinely a substitute for L1 work. It does not need the analyst's continued presence; it needs only the analyst's training data. Once that training is done, the L1 is operational overhead. Every L1 analyst who has watched an autonomy pitch in 2025 or 2026 has had this thought, and the thought is correct about *that* model of AI.

**The L1 analyst should fear opaque AI; this paper describes its opposite.** A Learning SOC built on passive capture and typed handoffs makes L1 work *more visible*, *more attributable*, and *more credit-traceable* — not less. The L1 whose dismissed alerts disappear into a black box is replaceable by anyone (or anything) that can also dismiss alerts. The L1 whose dismissed alerts become typed handoffs, whose ruled-out hypotheses become part of the team's doctrine, whose pattern recognition becomes suppression rules with their name on the audit trail, and whose intuition becomes free-form entries that future analysts query against — *that* L1 is the source of the institutional value the team accumulates. They are harder to replace, not easier.

The architectural pivot is real and worth naming directly. In the autonomous-SOC model, the L1 is a *sensor* whose output the platform consumes. In the Learning SOC model, the L1 is an *author* of doctrine whose reasoning the platform preserves and propagates. A sensor is a commodity; an author is a contributor. The first can be replaced; the second is what makes the team distinct.

**Three concrete things change for the L1 in a Learning SOC.**

First, their work becomes visible. The dismissed alerts that used to vanish into closed-ticket archives now show up in the team's doctrine as "patterns L1 analyst X identified as routine FP — confirmed by 47 subsequent cases over six months". The contribution is named, dated, and credit-traceable in a way that survives turnover and feeds performance review with evidence rather than impression.

Second, their judgment becomes reusable. The ruled-out hypothesis that today lives in an L1's head and disappears the day they leave becomes a typed entry that the next L1 inherits as starting context on similar cases. The senior analyst the team currently depends on as institutional memory is partly relieved of that role; the doctrine carries it. New L1 hires ramp faster because they inherit working judgment, not just procedures.

Third, their fear shifts targets. The L1 in a Learning SOC has a defensible argument against AI substitution: *"the system can't replace me because the system depends on what I uniquely produce — the typed reasoning that becomes the next case's starting context. The day the AI generates that without me is the day the AI is doing my job, not the day my job disappears into the AI."* This is not a guarantee against AI advancement, but it is a meaningfully different position than the L1 working against opaque autonomy who has nothing to point at as their non-substitutable contribution.

**This is also a contract a manager can offer honestly.** Adopting a Learning SOC means committing to a specific model of analyst work: *"your reasoning is the asset we are building; the platform exists to preserve and propagate it; your visibility is your insurance against opaque AI."* That contract is not free — it requires the manager to actually use the visibility (cite L1 contributions in performance reviews, attribute doctrine to its authors, treat the doctrine the team builds as the team's collective product), not just to enable it technically. Without that follow-through, the L1's fear stays warranted and adoption stalls in the second quarter.

The summary the L1 analyst can carry into Monday morning: *opaque AI replaces the analyst who executes; the Learning SOC values the analyst who thinks.* The first sentence is a threat. The second is a position the L1 can occupy with confidence — provided the platform genuinely makes the second visible and the manager genuinely treats it as the team's product.

A SOC manager planning to introduce typed handoffs and passive capture should not present the change as an efficiency program. Efficiency programs feel like the prelude to layoffs. The honest framing is the opposite: *"we are building a system in which your reasoning becomes the team's institutional asset, attributable and reusable; you are no longer the disposable layer above the alert queue, you are the source of the doctrine the team operates by."* That framing is true if the architecture is honestly implemented, and it is the only framing under which adoption survives the second high-pressure week.

### 12.1 What visibility means for the analyst who is not the strongest

There is a second fear, less often stated but real: visibility is comforting for the strong analyst and uncomfortable for the analyst who knows their work is mediocre. *"You are going to see exactly how I think and where I am wrong"* is a different threat than *"you are going to replace me with AI"*, and it lands on a different population of L1s — the ones who have been quietly under-performing in the cover that opaque ticket-closure provides. They are not wrong to feel the change.

The honest answer is in three parts.

First, the visibility this model creates is *contribution-positive at the individual level and aggregate-only at the comparative level*. The architectural guardrails inscribed in §11.4 refuse to render per-analyst slicing of the cognitive loss coefficient, bounce-back rate, or any of the operational signals. The team's reconstruction cost is visible at team and workflow level; the comparative ranking of *"analyst A vs analyst B on these metrics"* is structurally not a feature. Visibility on individual analysts is what they contributed (the doctrine entries authored, the suppression rules attributed, the reused reasoning), not what they cost relative to peers.

Second, the personal-engagement metrics that *are* per-analyst — the accept-vs-correct rate on AI proposals, the outcome track record of past validations — are visible to the analyst themselves first, by default visible to no one else. If a coach or manager sees them, it is because the analyst has chosen to share them in the context of voluntary skill development. The system supports the analyst's own visibility on their own reasoning quality before it supports anyone else's visibility on that analyst.

Third, and harder to say honestly: in a Learning SOC, the analyst who has been under-performing in the cover of opaque ticket-closure does eventually become visible — not through individual surveillance metrics, but through the absence of accumulated contribution. The L1 whose dismissals get reversed often, whose escalations are routinely thin, whose pattern recognition does not show up in the team's reused doctrine, will surface within a quarter or two regardless of what dashboard exists. This is not a punishment built into the platform; it is what happens when a team's reasoning becomes attributable. In an opaque SOC, that analyst could hide for years; in a Learning SOC, they cannot. The honest pitch is not that the visibility eliminates this — it is that the visibility makes it *actionable earlier*: structured coaching grounded in specific cases (*"here are five dismissals that were reversed; let's understand the pattern"*) replaces years of quiet failure followed by abrupt termination. For the analyst who can grow with structured feedback, this is a dramatically healthier outcome. For the analyst who genuinely belongs in a different role, earlier clarity is also healthier than slow attrition.

The analyst who hears this and feels exposed is not wrong. The argument is not that the exposure does not exist. The argument is that the exposure is *contribution-shaped, not surveillance-shaped*, that the per-analyst dashboards a manager could weaponize do not exist by architectural choice, and that the alternative — opaque mediocrity protected by the absence of measurement — is worse for the team, worse for the senior analysts who carry the weight, and ultimately worse for the under-performing analyst themselves. None of this softens the discomfort, but it places it honestly.

### 12.2 Architectural guardrails are necessary; manager culture and reciprocal visibility complete the picture

The architectural guardrails in §11.4 reduce the *surface* of surveillance that can be technically performed against an analyst. They do not, on their own, eliminate the *perception* of surveillance — and a SOC team that perceives itself as living in a panopticon will produce the same outcomes (resentment, bypass attempts, turnover) regardless of what the architecture technically permits. A platform that names only the architectural guardrails has solved the technical problem and left the social problem unsolved.

Three additional layers complete the picture, and they have to be inscribed alongside the architectural ones rather than left to manager discretion.

**Onboarding transparency.** From day one, an analyst working in a Learning SOC must know exactly what is captured, exactly what is surfaced and to whom, exactly what stays in their own personal view, and exactly what is structurally not renderable per-analyst. The transparency is itself a defense against perceived surveillance — uncertainty about what is being watched is a larger stressor than clarity about a smaller surveillance surface. Analysts who can read the data-model documentation and verify what the system can and cannot do are dramatically less stressed than analysts who guess.

**Right to query and contest one's own contribution data.** An analyst can request their full contribution history, see how it has been used, and contest entries they consider mis-attributed or unfairly characterized. The platform supports this as a first-class function, not as a buried administrative request. Ownership of the data starts with the analyst whose work it represents.

**Right to attribution anonymization on departure.** When an analyst leaves the team, the doctrine they contributed survives — that is the entire point of institutional memory — but their identifying attribution on those entries can be anonymized at departure on request. The doctrine entry continues to read *"contributed by [former team member]"* rather than naming the individual; the entry remains queryable, reviewable, and reusable, but it no longer functions as a perpetual employee profile that follows them after they leave. Total anonymization of all contribution data would defeat the model (doctrine must be attributable to be reviewable in real time by the team that uses it), but anonymization at departure threads the needle: institutional memory persists, individual perpetual exposure does not. Visibility-during-tenure is a fair trade for the contribution; surveillance-after-tenure is not, and the architecture should refuse it.

What this paper does *not* claim is any architectural mechanism that equalizes the manager-analyst power asymmetry. Audit logs that notify analysts when their data is queried, "reciprocal visibility" patterns, dashboards of who-watched-whom — none of these change the fundamental fact that the manager controls the analyst's employment and the analyst does not control the manager's. Pretending otherwise via tech-dressed mechanisms ("the analyst sees the gardien through the hatch") is what the paper refuses to do, because the pretense is what the critique fairly calls *gaslighting technologique*. The architecture does not balance power; it constrains the surveillance surface and supports analyst data rights, and stops there. Everything else lives in the manager-culture precondition (§9), where it actually is — not in an architectural illusion that shifts responsibility off the manager.

**Manager culture, named explicitly.** No combination of architectural guardrails and analyst-side rights survives a punitive manager. The same visibility deployed under a developmental manager produces coaching grounded in evidence; deployed under a punitive manager, it produces stress and attrition regardless of the architecture. The Learning SOC operating model assumes — and therefore must explicitly state — that the visibility it creates is intended for development, attribution, and team-level operational understanding, not for individual ranking, performance comparison, or punitive review. A manager who weaponizes the visibility is not implementing the model; they are implementing a different operating model under the same name. The platform should support this distinction by refusing to render the surveillance views (§11.4) and by making contribution-positive views the default surface, but the manager has to do the rest. A serious Learning SOC adoption involves explicit management training on the difference between contribution surfaces (intended) and performance ranking (out of scope and architecturally unsupported).

The honest summary: panopticon is not a technical risk that the architecture eliminates. It is a social risk that the architecture *constrains* — through the §11.4 guardrails, through onboarding transparency, through analyst data rights including departure-time anonymization — and that the manager must complete with a developmental rather than punitive culture. A platform that promises panopticon-immunity by architecture alone is over-claiming. A platform that names architecture, analyst rights, and manager culture together as the three required layers is honest about what visibility actually requires.

---

## 13. An Evaluation Agenda — Five Questions for the SOC Manager

Organizations that want to operationalize MTTU should begin with a small, concrete audit. The following five questions can be answered this week, by any SOC manager, with access to the team's last twenty-five escalations and one honest hour with the L2 analysts.

1. **Take the last five escalations.** For each, can the L2 act on the case without asking the L1 a clarifying question by chat, email, or in person? *(Tests the bounce-back rate that dashboards conceal.)*

2. **For the same five escalations:** can you reconstruct, from the case record alone — no chat, no analyst memory — what hypotheses L1 considered, what they ruled out, and why? If the answer relies on the L1 still being on the team, your operational memory is concentrated in a person, not in the system. *(Tests the senior-as-database failure mode.)*

3. **Audit your last twenty-five closed cases.** What fraction had material reasoning that lived in Slack, Teams, or email rather than in the case record? If it's above a quarter, your audit trail is incomplete and your ability to learn from past cases is constrained by chat retention policies. *(Tests the drift-to-chat failure mode.)*

4. **Time the last five L1→L2 transitions two ways:** the dashboard transition time (how fast did L2 acknowledge?), and the *first net-new analytical step* time (how long until L2 did something L1 had not already done?). The gap between the two is your hidden re-investigation cost. If it's larger than the dashboard transition, your fast escalation metric is masking duplicate work. *(Tests the silent re-investigation failure mode.)*

5. **If your most senior L1 takes a two-week vacation,** does L2 escalation throughput drop measurably? If yes, the team's continuity depends on a tacit decoding key one or two analysts hold. That is operational risk concentrated in two human memories. *(Tests the senior-as-database failure mode quantitatively.)*

A SOC that can answer all five honestly — and act on the answers — has the diagnostic baseline to know whether typed handoffs and passive capture are actually reducing the hidden cost. A SOC that cannot answer them is operating on dashboard signal rather than on operational reality.

## 14. Conclusion

MTTU highlights the hidden bottleneck in modern SOC operations: not detection speed, not response execution, but the transfer and reconstruction of understanding. The L1-to-L2 handoff is often where that bottleneck gets amplified.

If SOC teams want higher analytical throughput without lower standards, they need better transfer protocols, not just better dashboards. Structured, high-fidelity handoffs — composed by passive capture rather than written by tired analysts at the end of a shift — are one of the clearest ways to turn understanding from a per-analyst improvisation into a repeatable operational asset.

The architecture described in this paper has, in plain terms, three load-bearing parts. The **typed substrate** is the marrow — the clean, attributable data that every other layer reads and writes. **Passive capture** is the nervous system — recording the trace of work without asking the analyst to write a report at the end of a shift they have already survived. **Doctrine that refines through correction** is the brain — every L2 update of a thin handoff, every manager observation that lands as coaching, every senior review of a candidate pattern feeds back into the operational repertoire the team uses on the next case. Remove any one and the system stops being an organism and reverts to a filing cabinet with structured fields.

The honest pitch this paper makes is not that analysts will become geniuses, that handoffs will become perfect, or that AI will learn to investigate alone. It is narrower and harder to refute: **a SOC built on this architecture makes a given mistake only once.** The first L1 who under-considers lateral movement on an OAuth chain produces a thin handoff; the L2 corrects it; the doctrine absorbs the correction; the next analyst working a similar case starts with the better hypothesis already framed. That is what MTTU is fundamentally about — not the time it takes one analyst to think, but the time it takes the *organization* to convert a failure into knowledge that prevents the same failure from costing the same time again. That conversion interval, compressed from quarters to weeks, is what compounds.

The next paper in this series, *Typed Decision Bundle Chain*, formalizes the multi-stage version of this contract: how the same typed-handoff discipline scales beyond L1→L2 to cover triage, investigation, response, and incident closure as one continuous chain of preserved understanding.

---

## Terms Introduced

This paper uses or canonizes the following vocabulary. Some terms are inherited from *The Learning SOC* foundational paper; the new ones are marked with *(MTTU paper)*.

- **Mean Time To Understand (MTTU)** *(MTTU paper)* — the conceptual lens for the interval between alert assignment and the point at which the analyst has assembled sufficient context to make a defensible decision. Decomposes into context assembly, evidence review, hypothesis evaluation, and decision documentation. Not a stopwatch metric — see *Cognitive Loss Coefficient* for the measurable expression.
- **Cognitive Loss Coefficient** *(MTTU paper)* — the headline measurable signal: the fraction of L2 actions on a case that duplicate actions L1 already performed. A coefficient at 60% means roughly 40% of senior analyst budget is spent on reconstruction that produced no new analytical step. The single dashboard line that translates the MTTU lens into cash terms.
- **Lossy handoff** *(MTTU paper)* — an escalation transfer in which the receiving tier inherits less than the analytical state the sending tier produced, and must reconstruct the missing fraction. The default state of free-text ticket-based escalations.
- **Typed handoff** *(MTTU paper)* — an escalation transfer in which the analytical state — active hypotheses, ruled-out alternatives, evidence reviewed, open questions, escalation rationale — is preserved in stable structured slots that the receiving tier can read in minutes instead of rebuilding in hours. Carries both a structured layer and a free-form intuition layer.
- **Engaged verification** *(MTTU paper)* — verification calibrated to case stake. Cheap on routine cases by design; explicitly engaged (challenge questions, required engagement, optional co-review) on high-stakes or novel cases. The architectural answer to the rubber-stamping risk that cheap verification creates if applied uniformly.
- **Bounce-back** *(MTTU paper)* — an escalation that L2 returns to L1 because the handoff was too thin to act on. Round-trip cost rarely captured by dashboards.
- **Silent re-investigation** *(MTTU paper)* — an escalation that L2 absorbs without bouncing back, redoing L1's work invisibly because raising the issue is socially costly. The most common failure mode and the hardest to see in metrics.
- **Reasoning vacuum** *(MTTU paper)* — an escalation whose case record carries the status change but no record of the reasoning that produced it. Audit trail goes cold at the boundary.
- **Analyst as author** *(MTTU paper)* — the L1 role under a Learning SOC: contributor of typed reasoning that becomes part of the team's doctrine, with attribution and credit-trace. Distinct from "analyst as sensor" (the L1 role under autonomous-SOC models, where the platform consumes the analyst's output without preserving authorship). The architectural basis for the L1's adoption argument against opaque AI substitution.
- **Senior-as-database** *(MTTU paper)* — operational continuity concentrated in one or two senior analysts whose tacit knowledge is the only key to decoding the team's escalations. Visible as throughput drops during their absences.
- **Passive capture** *(foundational paper)* — recording of analyst reasoning as the typed trace of work itself, without explicit forms or end-of-case reporting. The architectural alternative to discipline-based handoff capture.

---

## References

Chuvakin, A. and Ghanizada, I. (2021). Autonomic Security Operations: 10X Transformation of the Security Operations Center.

Google Cloud. (2025). Agentic AI for Security Operations.

Klein, G. (1998). Sources of Power: How People Make Decisions.

Microsoft. (2026). The agentic SOC—Rethinking SecOps for the next decade.

Palo Alto Networks. (2025). Cortex AgentiX.

Shannon, C. E. (1948). A Mathematical Theory of Communication.

Starmer et al. (2014). Changes in medical errors after implementation of a handoff program.

Weick, K. E. (1995). Sensemaking in Organizations.