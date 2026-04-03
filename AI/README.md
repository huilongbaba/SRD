# Memoket AI Backend -- Software Requirements Document

> Version: 1.0.0
> Date: 2026-04-02
> Source: Architecture docs (L1-L5) + CodeBase/AI codebase analysis

## Document Map

| # | File | Scope |
|---|------|-------|
| 1 | [overview.md](overview.md) | Objective, scope, non-scope, tech stack |
| 2 | [system-context.md](system-context.md) | System boundary, interfaces, external dependencies |
| 3 | [modules/01-transcription.md](modules/01-transcription.md) | ASR pipeline (multi-provider routing, embedding) |
| 4 | [modules/02-summary-generation.md](modules/02-summary-generation.md) | Summary pipeline (parallel template gen, language handling) |
| 5 | [modules/03-ai-chat.md](modules/03-ai-chat.md) | Per-recording chat (InFile + CrossFile, SSE streaming) |
| 6 | [modules/04-insight-rag.md](modules/04-insight-rag.md) | RAG pipeline (pgvector + trigram + RRF fusion) |
| 7 | [modules/05-template-engine.md](modules/05-template-engine.md) | Template recommendation (topic-based, 3-tier fallback) |
| 8 | [modules/06-knowledge-graph.md](modules/06-knowledge-graph.md) | Cross-conversation relationship graph |
| 9 | [modules/07-voiceprint.md](modules/07-voiceprint.md) | Speaker identification (voiceprint enrollment + matching) |
| 10 | [modules/08-noise-reduction.md](modules/08-noise-reduction.md) | AI denoising (GTCRN model) |
| 11 | [modules/09-web-browsing.md](modules/09-web-browsing.md) | Web search for Agent mode (Serper API + page fetching) |
| 12 | [nfr.md](nfr.md) | Non-functional requirements (quantified from code) |
| 13 | [appendix/celery-tasks.md](appendix/celery-tasks.md) | Celery task chain specification |
| 14 | [appendix/prompt-spec.md](appendix/prompt-spec.md) | Key LLM prompt templates |

## Conventions

- **FR-XX-NNN**: Functional requirement ID (module prefix + sequence)
- **NFR-XX-NNN**: Non-functional requirement ID
- Priorities: **MUST** = mandatory; **SHOULD** = expected; **MAY** = optional
- Verification: **T** = unit/integration test; **I** = inspection/code review; **D** = demo; **A** = analysis
