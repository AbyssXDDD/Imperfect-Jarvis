# ARCHITECTURE.md — Imperfect-Jarvis

> This is the authoritative technical reference for Imperfect-Jarvis. Every new capability, plugin, or subsystem should be checked against this document before implementation. When a design decision here needs to change, record *why* in `docs/decisions/` (an ADR) rather than silently editing this file.

---

## 1. Engineering Philosophy

Imperfect-Jarvis is **intent-first**, not command-first and not automation-first.

| Model | How it thinks | Example failure |
|---|---|---|
| Chatbot-first | "Respond to what was said" | Can't act on the world |
| Automation-first | "Match trigger → run script" | Brittle; breaks on phrasing |
| **Intent-first (this project)** | "Infer the user's goal, then decide how to satisfy it" | Requires more upfront design, but generalizes |

Concretely: the input `"Hey Jarvis, open Discord"` is not routed directly to a Discord-launching function. It is first resolved to an **intent** (e.g. `SocialConnect { target: "Discord" }`), which a capability then decides how to fulfill — check if Discord is running, launch it if not, join the expected voice channel, report failure if it can't.

Design corollaries:
- Every new capability should integrate into the existing architecture, not bypass it.
- Prefer reusable services over one-off implementations.
- Favor maintainability and extensibility over shipping a feature quickly.
- The system should still make sense after years of incremental growth without a rewrite.

---

## 2. High-Level System Diagram

```
                     ┌─────────────────────────┐
                     │        Jarvis.App        │  (WinUI 3 shell, tray, overlay)
                     └────────────┬─────────────┘
                                  │
                     ┌────────────▼─────────────┐
                     │      Input Adapters       │  Voice (wake word/STT), Text, Hotkey
                     └────────────┬─────────────┘
                                  │  raw input
                     ┌────────────▼─────────────┐
                     │      Intent Resolver      │  raw input → structured Intent
                     │   (uses AI Provider)      │
                     └────────────┬─────────────┘
                                  │  Intent
                     ┌────────────▼─────────────┐
                     │   Dispatcher (Mediator /  │  routes Intent → Capability
                     │   Event Bus, see §6)      │
                     └────────────┬─────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
      ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
      │  Capability A  │   │  Capability B  │   │  Capability C  │  (Core-defined contracts)
      └───────┬───────┘   └───────┬───────┘   └───────┬───────┘
              │ implemented by            implemented by
      ┌───────▼───────┐                          ┌───────▼───────┐
      │ Plugin: Discord │                        │ Plugin: Spotify │  (Infrastructure, hot-swappable)
      └────────────────┘                          └────────────────┘

      Cross-cutting, available to all layers above:
      ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐
      │  AI Provider   │  │    Memory      │  │  Permissions   │  │  Vision Svc   │
      │  Abstraction   │  │    System      │  │    System      │  │ (screen aware)│
      └───────────────┘  └───────────────┘  └───────────────┘  └───────────────┘
```

---

## 3. Solution & Folder Structure

Clean Architecture, dependency direction always points inward (Infrastructure/Plugins → Application → Core; never the reverse).

```
Imperfect-Jarvis/
├── src/
│   ├── Jarvis.Core/                 # Entities, interfaces, no dependencies on anything else
│   │   ├── Capabilities/            # ICapability and capability contracts
│   │   ├── Intents/                 # Intent models
│   │   ├── AiProviders/             # IAiProvider interface
│   │   ├── Memory/                  # Memory abstractions
│   │   ├── Permissions/             # Permission model
│   │   └── Events/                  # Event / message contracts
│   │
│   ├── Jarvis.Application/          # Use cases, orchestration logic, depends only on Core
│   │   ├── IntentResolution/
│   │   ├── Dispatch/                # Mediator / event bus implementation
│   │   └── Routines/                # "gaming mode" style composite routines
│   │
│   ├── Jarvis.Infrastructure/        # Concrete implementations, depends on Application+Core
│   │   ├── AiProviders/
│   │   │   ├── Ollama/
│   │   │   ├── OpenAi/
│   │   │   ├── Anthropic/
│   │   │   └── Gemini/
│   │   ├── Memory/
│   │   │   ├── Sqlite/
│   │   │   └── VectorStore/
│   │   ├── Vision/
│   │   ├── Automation/              # Windows APIs, PowerShell, UI Automation
│   │   └── Security/                # Encrypted secret storage, permission enforcement
│   │
│   ├── Jarvis.Plugins.Sdk/          # Public SDK surface third-party/first-party plugins build against
│   │
│   ├── Jarvis.Plugins.*/            # One project per plugin (Discord, Spotify, Steam, ...)
│   │
│   └── Jarvis.App/                  # WinUI 3 shell — composition root, DI wiring, tray icon, overlay UI
│
├── docs/
│   ├── architecture/
│   │   ├── ARCHITECTURE.md          # this file
│   │   ├── PROJECT.md
│   │   ├── CONSTITUTION.md
│   │   ├── FEATURES.md
│   │   ├── ROADMAP.md
│   │   └── CLAUDE.md
│   ├── design/
│   ├── api/
│   └── decisions/                   # ADRs — one file per significant decision
│
└── tests/
    ├── Jarvis.Core.Tests/
    ├── Jarvis.Application.Tests/
    └── Jarvis.Infrastructure.Tests/
```

---

## 4. Capabilities & Plugins

- A **Capability** is a contract defined in `Jarvis.Core` describing *what* can be done (e.g. `IMessagingCapability.JoinVoiceChannel`).
- A **Plugin** is a concrete implementation of one or more capabilities, living in its own `Jarvis.Plugins.*` project.
- This is not a new concept bolted onto Clean Architecture — it *is* Clean Architecture's Core/Infrastructure split, given domain-appropriate names so the plugin SDK reads clearly to third parties later.

### Plugin lifecycle
```
Discovered → Validated (manifest, permissions declared) → Loaded → Registered (capabilities exposed to Dispatcher) → Enabled/Disabled (runtime toggle) → Unloaded
```

### Plugin manifest (conceptual)
Every plugin declares, at minimum:
- Unique ID, version, display name
- Capabilities it implements
- Permissions it requires (see §7) — requested, never assumed
- Compatibility range (min/max Jarvis core version)

### Marketplace readiness
No plugin store ships in v1. But the manifest format, versioning, and load/unload lifecycle above are designed so a `Jarvis.Plugins.Store` project can be added later without changing the plugin contract.

---

## 5. AI Provider Abstraction

Jarvis never calls Ollama (or any model) directly from application logic.

```
Jarvis.Application
        │
        ▼
  IAiProvider  (Core interface: Complete(), Stream(), Embed())
        │
   ┌────┼────┬─────────┬───────────┐
   ▼    ▼    ▼         ▼           ▼
Ollama OpenAI Anthropic Gemini  (future providers)
```

- Default/local provider: **Ollama**, selected because it's local-first and free to run.
- Cloud providers are equally first-class citizens of the interface, not bolted-on exceptions — selectable per-request or per-routine (e.g. "use a cloud model for this because it needs stronger reasoning").
- `IAiProvider` must expose an `Embed()` method from day one, even though nothing consumes it until §6.3 (semantic memory) — Ollama can serve local embedding models (e.g. `nomic-embed-text`), keeping the whole pipeline local-first if desired.
- **Open decision, not yet locked:** which local embedding model and which vector index (e.g. a lightweight local vector DB vs. an in-SQLite extension). Record the choice as an ADR when made.

---

## 6. Dispatch: Mediator Now, Event Bus Later

**Decision:** start with an in-process **mediator** pattern (a single dispatcher routing `Intent → Handler`), not a full pub/sub event bus, despite the instinct to decouple everything immediately.

Rationale: a real event bus is itself nontrivial infrastructure (topic design, delivery guarantees, debugging story). With 0–2 plugins in Milestone 0–2, there is nothing yet to decouple, and building it early means designing it speculatively rather than from felt pain.

**The migration path is designed in from day one:**
- `Jarvis.Application.Dispatch` defines `IDispatcher` and `INotificationPublisher` as interfaces now.
- The mediator implementation satisfies both trivially (in-process, synchronous-ish).
- When plugin count and cross-plugin coordination justify it (see ROADMAP phase where multiple automation plugins coexist), swap the implementation behind the same interfaces for a real event bus — no call sites need to change.

Target shape once graduated:
```
Voice → Intent → Event → Plugin(s) subscribed to that event
```
rather than
```
Voice → (hardcoded) → Discord Plugin
```

---

## 7. Permission System

"Respects permissions" is a value statement, not a design. This is the actual model.

### Model (Android-style, per-capability)
| Capability example | Default posture |
|---|---|
| Read clipboard | Allowed |
| Launch Discord | Ask once, then remember |
| Delete files | Always ask |
| Run PowerShell as current user | Ask, show exact command |
| Run PowerShell as Administrator | Always ask, explicit elevation dialog, never silently escalate |
| Continuous screen observation | Always ask, session-scoped, revocable mid-session |

### Rules
1. A plugin declares required permissions in its manifest at load time — nothing is requested silently mid-run.
2. Grants are scoped to a capability, not to a whole plugin, so a plugin needing both "read files" and "delete files" can be granted one without the other.
3. Destructive or irreversible actions (delete, elevate, financial, outbound message send) are **always-ask**, never "remember this choice," regardless of prior grants.
4. All grants are visible and revocable from a settings surface — this is a hard requirement, not a nice-to-have, given the automation surface area (§9) includes PowerShell and file deletion.
5. Every permission check and grant/deny decision is logged (see §11).

---

## 8. Memory Architecture

Memory is a tiered system, not a single SQLite table — a flat store is how this becomes "just another chatbot."

```
┌─────────────────────────────────────────────────┐
│                  Memory System                   │
├───────────────┬───────────────┬─────────────────┤
│  Short-term    │   Long-term    │   Semantic       │
│  (conversation │  (durable      │   (embeddings,   │
│   context,     │   facts:       │    retrieved by  │
│   volatile,    │   preferences, │    similarity,   │
│   session-     │   routines,    │    e.g. "what    │
│   scoped)      │   projects)    │    did I say     │
│                │                │    about X       │
│                │                │    last month?") │
└───────────────┴───────────────┴─────────────────┘
```

- **Short-term:** in-memory conversation window, cleared/summarized at session boundaries.
- **Long-term (structured):** SQLite-backed — preferences, known routines, known projects (this is where "Imperfect Studio," "Immortal Paradox," etc. would live if the user references their own projects to Jarvis).
- **Semantic (unstructured):** vector-indexed embeddings of past interactions/documents, enabling similarity retrieval rather than exact lookup. Depends on the `Embed()` decision in §5.
- Retrieval into a live conversation should prefer long-term structured facts first (cheap, precise), falling back to semantic search only when there's no structured hit.

**Explicit open questions for CONSTITUTION.md / ADRs:**
- Retention policy — does anything ever get forgotten, or does the user need an explicit "forget X" command?
- Where do secrets/PII intersect with memory (e.g. should Jarvis remember a password it saw on screen)? This connects directly to §9's vision safeguards.

---

## 9. Vision / Screen Understanding

This is the highest-risk capability in the system and needs constraints stated explicitly, not left to "user enables it."

```
Desktop → Vision Service → (local processing preferred) → AI Provider (only if needed)
```

### Non-negotiable constraints
1. **On-device by default.** Screenshots/frames are processed locally unless the user has explicitly opted a *specific session* into cloud vision — this is a session-scoped grant per §7, not a global setting toggled once.
2. **No silent continuous capture.** "Observe the desktop continuously" is a distinct, higher-tier permission from "take one screenshot when asked" — these must not share a single permission flag.
3. **Sensitive region awareness.** Where feasible, password fields and known-sensitive UI regions (browser credential managers, terminal sessions typing secrets) should be excluded or masked before any frame leaves local processing.
4. Vision results are treated as short-term memory by default (§8) — they are not persisted to long-term/semantic memory unless the user explicitly says to remember something seen on screen.

---

## 10. Personality System

Personality is configuration, not a single hardcoded system prompt.

| Axis | Example values |
|---|---|
| Humor | none / dry / playful |
| Formality | casual / neutral / formal |
| Response length | terse / normal / detailed |
| Wake word | "Hey Jarvis" (configurable) |
| Voice | selectable TTS voice |
| Routine vocabulary | user-defined aliases, e.g. "goon mode" → a specific routine definition |

These parameters feed into intent resolution and response generation as structured configuration, not by string-concatenating a prompt — this keeps the personality system swappable and testable independent of which `IAiProvider` is active.

---

## 11. Security, Logging & Diagnostics

- **Secrets** (API keys, tokens) — encrypted at rest, never in plaintext JSON config, never logged.
- **Logging** — structured logs for: every permission grant/deny, every plugin load/unload, every PowerShell/automation command executed (command text + outcome), every AI provider call (provider + latency, not necessarily full content by default).
- **Diagnostics surface** — a visible log/activity view in-app, since a system with this much automation surface needs to be auditable by the person running it, not just by a developer with file access.

---

## 12. Sequencing (ties to ROADMAP.md)

Architecture above is designed to support the long-term vision, but implementation is deliberately staged so nothing above is fully built before it's needed:

1. **Core platform** — Core/Application projects, mediator (§6), AI provider abstraction (§5) with a trivial echo provider
2. **Milestone 0: walking skeleton** — `Jarvis.App` boots, DI wired, "Jarvis is alive"
3. **Text chat** — real Ollama connection through `IAiProvider`
4. **One automation plugin** — e.g. launch an application — proves the Capability/Plugin split end-to-end
5. **Voice input/output**
6. **Memory** (start with long-term structured tier only; semantic tier once retrieval is actually needed)
7. **Vision** (with §9 constraints built in from its first line of code, not retrofitted)
8. **Advanced automation** (PowerShell, file management, multi-app orchestration)
9. **Proactive behavior**

Each phase should produce something runnable. Nothing in §5–§11 is fully built before the phase that needs it — but every interface exists early enough that later phases slot in rather than requiring rework.
