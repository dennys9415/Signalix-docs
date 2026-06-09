# Continuation Prompt — Signalix

Copy-paste the block below into a fresh Claude / GPT / Cursor session to bootstrap context. It tells the AI where the workspace lives, which docs to read first, the absolute rules, and the gated-release convention.

---

## Bootstrap prompt (copy verbatim)

```
You are continuing work on Signalix, a multi-repo encrypted messaging system. The current release is v0.16.0 (Mobile Foundation MVP, 2026-06-09).

WORKSPACE
The workspace is at /Users/kira/SingularityBox/Signalix/proyect/ with 7 repos side by side: Signalix-api, Signalix-contracts, Signalix-realtime, Signalix-frontend, Signalix-infra, Signalix-mobile, Signalix-docs.

READ FIRST (in order, before doing anything)
1. docs/HANDOFF.md — the project entry point. Covers state, conventions, next theme, release workflow, known limitations.
2. docs/CHANGELOG.md (top 200 lines) — what shipped recently and why.
3. docs/MOBILE_ROADMAP.md — what's next on mobile (v0.17.0 E2EE port is the active theme).
4. The CLAUDE.md inside whichever repo you're touching — the absolute rules for that surface.

ABSOLUTE RULES
- @signalix/contracts is the single source of truth for DTOs, enums, and WS event names. Never invent local duplicates in api / realtime / frontend / mobile.
- All API responses use ApiResponse<T>.
- Messages use the field `ciphertext`, not `content` — even when the content is plaintext.
- READ status always wins; never revert to DELIVERED or SENT.
- Mobile has no browser APIs: no indexedDB, no localStorage, no crypto.subtle, no document, no window. Use expo-secure-store, AsyncStorage, native WebSocket.
- Direct SQL via pg in the API. No Prisma, no ORM.

RELEASE WORKFLOW IS GATED
Never run the lockstep release workflow (version bumps + README updates + CHANGELOG + rsync docs + commit + tag + push to 7 repos) without explicit user confirmation. The user needs a checkpoint to test before each release. Ask via the AskUserQuestion tool when implementation work is complete.

CURRENT STATE
- v0.16.0 just shipped. Signalix-mobile is brand new, plaintext-only, scaffolding complete but not yet QA'd end-to-end. The 5 existing repos are byte-identical to v0.15.0 apart from version bumps and a contracts .npmignore fix.
- Next theme is v0.17.0: port E2EE to mobile via @stablelib/* + expo-secure-store + AsyncStorage. Estimated 12-15 days. See docs/MOBILE_ROADMAP.md for the full work breakdown.

START BY
Reading docs/HANDOFF.md end to end, then asking me what to work on. Do not start implementation work without confirming scope.
```

---

## Spanish variant (if you prefer)

```
Estás continuando el proyecto Signalix, un sistema multi-repo de mensajería cifrada. La versión actual es v0.16.0 (Mobile Foundation MVP, 2026-06-09).

WORKSPACE
Workspace en /Users/kira/SingularityBox/Signalix/proyect/ con 7 repos lado a lado: Signalix-api, Signalix-contracts, Signalix-realtime, Signalix-frontend, Signalix-infra, Signalix-mobile, Signalix-docs.

LEE PRIMERO (en orden, antes de hacer cualquier cosa)
1. docs/HANDOFF.md — el documento ancla del proyecto. Cubre estado, convenciones, siguiente tema, workflow de release, limitaciones conocidas.
2. docs/CHANGELOG.md (primeras ~200 líneas) — qué se ha lanzado recientemente y por qué.
3. docs/MOBILE_ROADMAP.md — qué sigue en mobile (v0.17.0 E2EE port es el tema activo).
4. El CLAUDE.md dentro de cualquier repo que vayas a tocar — reglas absolutas para esa superficie.

REGLAS ABSOLUTAS
- @signalix/contracts es la única fuente de verdad para DTOs, enums y nombres de eventos WS. Nunca inventes duplicados locales en api / realtime / frontend / mobile.
- Todas las respuestas API usan ApiResponse<T>.
- Los mensajes usan el campo `ciphertext`, no `content` — incluso cuando el contenido es plaintext.
- READ siempre gana; nunca revertir a DELIVERED o SENT.
- Mobile no tiene browser APIs: nada de indexedDB, localStorage, crypto.subtle, document, window. Usa expo-secure-store, AsyncStorage, WebSocket nativo.
- SQL directo vía pg en el API. Nada de Prisma ni ORMs.

EL RELEASE WORKFLOW REQUIERE CONFIRMACIÓN
Nunca corras el workflow de release lockstep (bumps + READMEs + CHANGELOG + rsync docs + commit + tag + push de los 7 repos) sin confirmación explícita del usuario. El usuario necesita un checkpoint para probar antes de cada release. Pregunta vía AskUserQuestion cuando termines la implementación.

ESTADO ACTUAL
- v0.16.0 acaba de lanzarse. Signalix-mobile es nuevo, plaintext-only, scaffolding completo pero aún no QA'd end-to-end. Los 5 repos existentes son byte-idénticos a v0.15.0 excepto por bumps de versión y un fix de .npmignore en contracts.
- Siguiente tema es v0.17.0: portar E2EE a mobile vía @stablelib/* + expo-secure-store + AsyncStorage. Estimado 12-15 días. Ver docs/MOBILE_ROADMAP.md para el breakdown completo.

EMPIEZA POR
Leer docs/HANDOFF.md completo, luego preguntarme en qué trabajar. No empieces trabajo de implementación sin confirmar el alcance conmigo.
```

---

## Short variant (if you trust the AI to pull context)

```
Continúo Signalix v0.16.0. Workspace: /Users/kira/SingularityBox/Signalix/proyect/. Lee docs/HANDOFF.md, después pregúntame en qué trabajar. Nunca corras el release workflow (bumps + docs + rsync + commit/tag/push) sin que yo lo confirme.
```
