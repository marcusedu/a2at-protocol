# A2AT: App-to-Agent Tunnel
## A Proposal for Ephemeral, Scoped, and Auditable Delegation Between Applications and User-Controlled AI Agents

**Status of this Document:** Informational / Draft Proposal (pre-standardization)
**Category:** Experimental
**Author:** [Your Name] — [Your Affiliation / Independent]
**Date:** July 2026
**Version:** 0.1 (draft)

---

## Abstract

This document proposes A2AT (App-to-Agent Tunnel), an application-layer protocol for third-party applications to request bounded, auditable assistance from a user's own AI agent — without the application holding a model-provider API key, and without the agent's provider treating the request as anonymous, unscoped automation. A2AT defines: (1) an ephemeral, forward-secure session-establishment handshake; (2) a scoped, human-readable consent model enforced at the agent layer rather than the application layer; (3) a tamper-evident audit log co-signed by both parties; (4) application attestation to prevent consent-phishing by spoofed clients; and (5) mitigations for confused-deputy and aggregation-based privacy attacks. A2AT is explicitly designed to be adopted by AI providers as a first-party capability — analogous to how OAuth is implemented by identity providers, not bolted on by relying parties — rather than as a workaround built by applications against provider terms of service.

---

## 1. Introduction and Motivation

### 1.1 The problem

Independent developers who want to add AI features to an application face an uncomfortable choice:

1. **Embed a provider API key** (their own or a shared backend key). This creates a single point of financial and security failure: a leaked key can generate unbounded usage charges before the provider's fraud detection intervenes, and the developer bears full liability regardless of who caused the leak.
2. **Ask each user to bring their own API key** (BYOK). This shifts financial risk to the user and is currently the most defensible production pattern, but it still requires every user to independently create a developer account with a model provider, generate a key, and manage its lifecycle — a meaningful barrier for non-technical users.
3. **Manual prompt relay.** The user copies a generated prompt from the application, pastes it into their own AI chat client (ChatGPT, Gemini, Claude, etc.), copies the structured response back, and pastes it into the application. This requires no key management at all, but it is slow, error-prone, and does not scale to any workflow that requires more than one round trip.

None of these options give the user what they actually have in a consumer chat AI subscription: an already-authenticated, already-paid-for reasoning agent that they trust and use daily. The gap is not model capability — it is a missing *transport and trust layer* between an arbitrary third-party application and a user's already-provisioned agent session.

### 1.2 Why this is not simply "reverse MCP"

The Model Context Protocol (MCP) and the OpenAI Apps SDK formalize the *agent calling the application* as a tool. A2AT addresses the inverse direction — the *application requesting the agent's help* — but it must not be built as an unofficial replay of a user's subscription session against a chat client's undocumented endpoints. That pattern already exists informally (CLI OAuth tokens routed through local proxies to reuse a subscription's quota in third-party tools) and has already been explicitly prohibited: Anthropic amended its terms in February 2026 to disallow subscription OAuth tokens in third-party tools, with billing enforcement activated in April 2026, and Google made an equivalent change to Gemini CLI access around the same time. This is not an oversight in the ecosystem that A2AT can quietly exploit — it is a deliberate boundary set by providers for both abuse-prevention and business-model reasons (subscription pricing assumes first-party usage patterns; automated third-party traffic does not fit that model).

A2AT is therefore scoped as a **standardization proposal**, not a client-side workaround. It defines the protocol surface a provider would need to expose for this pattern to be legitimate, auditable, and compatible with their existing quota and abuse-prevention systems — comparable to how "Sign in with Google" required Google to build and operate the identity provider side, not merely for relying parties to reverse-engineer Google's login form.

---

## 2. Terminology

- **App**: the third-party application requesting agent assistance.
- **Agent**: the user's AI client/runtime (a chat application, an on-device model, or a provider-hosted session) acting under the user's authority.
- **Provider**: the entity operating the Agent's underlying model and enforcing its terms of use (e.g., the company behind the chat client).
- **Principal**: the human user who owns both the App account and the Agent session.
- **Session**: a single bounded A2AT exchange between an App and an Agent, bracketed by explicit open/close events.
- **Scope**: a declared, machine-readable and human-readable statement of what data the App is requesting and what the Agent is authorized to do with it.
- MUST / SHOULD / MAY are used per RFC 2119 to indicate requirement levels for a conformant implementation.

---

## 3. Threat Model

A2AT assumes the following adversaries and explicitly does **not** assume protection against all of them — each is addressed or acknowledged as an accepted limitation:

| Adversary | Capability | Addressed by A2AT? |
|---|---|---|
| Network eavesdropper (passive/active MITM) | Can observe or inject traffic between App and Agent | Yes — session-bound ephemeral keys, replay protection |
| Malicious/spoofed App | Impersonates a legitimate App to phish consent or exfiltrate data | Yes — App attestation |
| Compromised App (legitimate but exploited) | Legitimate App sends adversarial prompts to over-reach scope | Partially — confused-deputy mitigations, scope enforcement |
| Malicious/compromised Agent client | The Agent itself misbehaves or is compromised on-device | **Not addressed** — out of scope; endpoint security is the Principal's and Provider's responsibility |
| Provider itself | The Provider can observe all traffic it routes and may retain data per its own policies | **Not addressed** — A2AT assumes the Principal already trusts the Provider by virtue of using its Agent; A2AT does not attempt to hide data from the Provider |
| Aggregating adversary | A single App makes many individually-authorized requests to reconstruct a fuller profile than any single grant implies | Yes — aggregation-aware consent accounting |

Being explicit about the last two rows matters: A2AT is a trust-relay protocol between App and Principal, mediated by an Agent the Principal already trusts. It is not a confidentiality protocol against the Provider, and it cannot substitute for device-level security.

---

## 4. Architecture Overview

```
 ┌────────┐   1. Session Request (signed, scoped)   ┌───────────────┐
 │  App   │ ───────────────────────────────────────▶│     Agent      │
 │        │                                          │ (user-trusted) │
 │        │◀─────────────────────────────────────────│               │
 └────────┘   2. Consent UI shown to Principal        └───────────────┘
      │                                                       │
      │  3. Ephemeral session key exchange (forward-secure)   │
      │◀──────────────────────────────────────────────────────
      │
      │  4. Scoped request/response exchange (one or more turns)
      │◀────────────────────────────────────────────────────▶│
      │
      │  5. Session close → key invalidation → audit record co-signed
      ▼
 ┌─────────────────────────┐
 │  Tamper-evident log      │  (held by Agent, visible to Principal)
 └─────────────────────────┘
```

The Agent — not the Provider's raw API — is the trust boundary. The App never receives a model-provider API key at any point; it only ever communicates with the Agent, which the Principal already trusts and which enforces scope and consent on the Provider's behalf.

---

## 5. Session Establishment

### 5.1 Ephemeral key exchange

A2AT MUST use an ephemeral key-agreement scheme providing forward secrecy — e.g., an ephemeral Elliptic-Curve Diffie–Hellman exchange per session, following the same design principle as the Signal/Noise Protocol family. A session key compromised after the session closes MUST NOT allow decryption of past session traffic.

### 5.2 Replay protection and key binding

Each session token MUST be sender-constrained, not merely bearer. This is not a novel requirement — it is the same problem OAuth 2.1's **DPoP** (Demonstrating Proof-of-Possession, RFC 9449) extension already solves: a token is bound to a private key held by the requesting party, so an intercepted token cannot be replayed by a third party who does not also hold that key. A2AT SHOULD reuse DPoP directly rather than defining a new binding mechanism.

### 5.3 Session closure

On explicit close (App-initiated, Agent-initiated, Principal-initiated, or timeout), all session keys MUST be invalidated Agent-side immediately, and any App-side copies become cryptographically useless due to forward secrecy — not merely "deleted by convention." An interceptor who captured session traffic gains nothing from replaying it after closure.

---

## 6. Scoped Consent Model

### 6.1 Consent must be granular, not binary

A single "Allow this app to access the Agent? Y/N" prompt is insufficient. A2AT requires scope declarations with, at minimum:

- **Data category** (e.g., purchase history, location, a specific document) — not a free-text justification alone.
- **Purpose** (one-time answer vs. ongoing/recurring access).
- **Retention** (does the Agent persist this exchange in its own history, and for how long).

### 6.2 Consent UI must resist manipulation, not just render text

Because the App supplies the justification text shown to the Principal, a malicious or careless App can write a persuasive but misleading justification — this is social engineering directed at the consent screen itself, distinct from prompt injection directed at the model. Implementations SHOULD structure the consent UI so that the **data category and scope are rendered by the Agent from a fixed, App-independent vocabulary** (a closed set of scope identifiers, similar to OAuth scopes), while the App's free-text justification is visually and semantically subordinate to that fixed label — never a substitute for it.

---

## 7. Audit Log Design

Every request/response pair transiting the tunnel MUST be recorded in a log that is:

- **Visible to the Principal** inside the Agent's own interface (not the App's).
- **Tamper-evident**: each entry includes the hash of the previous entry (hash-chained, as in a simplified Merkle/blockchain-style log), so retroactive edits are detectable.
- **Co-signed**: both App and Agent sign the record of what was requested and what was disclosed, giving both sides non-repudiation. Without co-signing, a compromised Agent client could unilaterally edit its own log to hide misbehavior, and an App could later dispute a legitimate log entry.

---

## 8. Application Attestation

Without proof of App identity, a cloned or spoofed App could open an A2AT session claiming to be a trusted, previously-authorized application, and phish consent under a familiar name. A2AT requires App attestation at session-open time using existing platform mechanisms — Android Play Integrity API, iOS App Attest, or equivalent code-signing verification — checked by the Agent before rendering any consent UI. This is not novel cryptography; it is a requirement that the protocol not skip a step every mobile platform already provides for exactly this purpose.

---

## 9. Revocation

Two distinct revocation cases must be handled differently:

1. **Session-scoped access** — resolved automatically at session close (Section 5.3).
2. **Standing/recurring grants** — the Principal must be able to revoke a previously granted recurring scope from the Agent's UI at any time, and revocation MUST propagate before the next request under that scope is served, not merely "eventually." Implementations SHOULD treat standing grants as short-lived and re-confirmable (e.g., re-prompt after N days or M requests) rather than indefinite.

---

## 10. Confused-Deputy Mitigations

This is the most consequential open problem in the design. A Principal's Agent typically has access to tools and data sources beyond what any single App is authorized to see (email, calendar, other connected apps via MCP). A malicious or compromised App can craft a request that, under the pretext of a legitimate purpose, induces the Agent to also draw on those *other* sources when composing its answer — laundering an out-of-scope data request through an in-scope-looking one.

A2AT requires that:

- The Agent's scope enforcement MUST be evaluated against the request's **declared scope identifiers**, not the App's free-text prompt content, and MUST reject or strip any part of a response that would require access outside the declared scope — even if the model itself was willing to produce it.
- Scope enforcement SHOULD happen as a hard filter on the Agent's tool-access layer (i.e., the model simply does not have the other tools available during an A2AT-scoped turn), not as a soft instruction to the model to "please stay in scope." Prompt-level instructions are not a security boundary.

This section should be flagged explicitly as an area needing further formal analysis before any implementation is considered production-ready.

---

## 11. Aggregation and Rate-Limiting

Individually-authorized requests can still be combined over time to reconstruct a broader profile than any single grant implies. A2AT recommends Agents track **cumulative disclosure per scope, per App, over a rolling window**, and re-trigger consent once a threshold is crossed — not just rate-limit by request volume.

---

## 12. Relationship to Existing Standards

A2AT is deliberately not a reinvention. It composes:

- **OAuth 2.1** for the authorization framing.
- **DPoP (RFC 9449)** for sender-constrained, replay-resistant tokens.
- **MCP / Apps SDK-style tool scoping** for how the Agent exposes and restricts tool access during a session.
- Standard **mobile platform attestation APIs** for App identity.

The contribution of A2AT is not new cryptography — it is defining the *consent, audit, and confused-deputy* semantics specific to the App→Agent direction, which none of the above standards address on their own.

---

## 13. Deployment Path and Provider Adoption

A2AT cannot be deployed unilaterally by an application against an unmodified chat client — this would reproduce the exact pattern providers have already moved to shut down (Section 1.2). A realistic adoption path requires:

1. A **reference implementation and formal write-up** (this document, iterated toward a fuller draft) circulated for feedback from developers already active in the MCP/Apps SDK ecosystem.
2. Engagement with at least one provider's developer relations or standards team, positioning A2AT as a complementary capability to MCP/Apps SDK (inverse direction) rather than a competing or circumventing mechanism.
3. A narrow, opt-in pilot — e.g., a provider-sanctioned scope type within an existing Apps SDK / extension surface — before proposing any broader standardization.

---

## 14. Security Considerations (Summary)

- Forward secrecy is mandatory; session compromise must not retroactively expose past traffic.
- Tokens must be sender-constrained (DPoP or equivalent), not bearer-only.
- Consent must be scope-labeled by a closed, App-independent vocabulary; App-supplied justification text is informational only, never authoritative.
- Scope enforcement must be a hard tool-access boundary, not a prompt instruction.
- Audit logs must be tamper-evident and co-signed by both parties.
- App identity must be attested via platform mechanisms before consent is requested.
- Cumulative disclosure must be tracked per scope to prevent aggregation attacks.
- A2AT does not protect against a compromised Agent endpoint, a compromised device, or the Provider's own data handling — these remain out of scope and must be stated as such in any implementation's documentation.

---

## 15. Open Questions and Future Work

- Formal verification of the confused-deputy mitigation (Section 10) using an existing protocol-analysis framework (e.g., Tamarin or ProVerif).
- Standardized scope vocabulary — who governs it, how is it extended, and how do providers keep it consistent across Agents.
- Recovery semantics when the Agent is offline or rate-limited by its own Provider mid-session.
- Whether A2AT sessions should be observable/auditable by the Provider itself for abuse detection, and what that implies for the "not a confidentiality protocol against the Provider" stance in Section 3.

---

## References

1. IETF RFC 9449 — *OAuth 2.0 Demonstrating Proof-of-Possession (DPoP)*
2. IETF RFC 6749 / OAuth 2.1 (draft) — *The OAuth Authorization Framework*
3. Anthropic — Terms of Service update on subscription OAuth tokens in third-party tools, February–April 2026
4. OpenAI — *Apps SDK: Guide to Authentication and User Consent*
5. Model Context Protocol (MCP) specification
6. Signal Protocol / Noise Protocol Framework — design rationale for forward-secure ephemeral key exchange

---

*This is a draft (v0.1) intended to solicit feedback before wider circulation. Comments, critiques, and prior-art pointers are welcome.*
