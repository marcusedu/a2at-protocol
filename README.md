# A2AT: App-to-Agent Tunnel

[Portuguese 🇧🇷](README-pt-br.md)

**Secure, ephemeral and user-controlled connection between third-party apps and personal AI agents.**

A proposal for a new protocol that lets any application request help from the user's own AI (ChatGPT, Claude, Gemini, etc.) **without sharing API keys**, without forcing users to manage keys, and with strong privacy & security guarantees.

---

## The Problem

As developers, we face an uncomfortable choice:

- Embed our own API key → **single point of failure** (I personally lost money when my key leaked).
- Ask users to bring their own key (BYOK) → high friction, especially for non-technical users.
- Manual copy & paste → works, but creates terrible user experience.

Meanwhile, millions of users already pay for powerful AI subscriptions. Why can't apps simply **ask the user's own agent for help** safely?

## The Solution: A2AT

**App-to-Agent Tunnel** is a protocol that creates a temporary, scoped, and auditable tunnel between your app and the user's personal AI agent.

### Key Features

- **No API keys** ever shared with the app
- **Ephemeral sessions** with forward secrecy
- **Granular scoped consent** shown by the Agent (not the app)
- **App attestation** to prevent spoofing
- **Tamper-evident co-signed audit log**
- **Strong protections** against prompt injection and confused deputy attacks
- Designed to be **provider-friendly** (complements MCP and A2A)

## Why This Matters

For **developers** (especially indie and small teams):
- Drastically reduce security and financial risks
- Offer AI features without heavy infrastructure costs
- Better user experience → higher conversion and retention

For **users**:
- Use the AI they already pay for and trust
- Full control and visibility over what the app can access
- No more manual copy-pasting

## Status

**Draft v0.1** — This is an open proposal seeking feedback from the community, AI providers, and developers.

[Read the full Whitepaper](./a2at-whitepaper.md)

## Get Involved

- ⭐ Star the repo if you find the idea valuable
- Share your thoughts and use cases
- Contribute to the protocol design
- Help spread the word with AI product builders and providers

---

**Built from real pain.**  
Let's create a better way to connect applications and personal AI agents.

---

**License**: MIT
