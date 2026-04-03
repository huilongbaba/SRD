# BACKEND SRD -- Memoket Backend

> Generated: 2026-04-02
> Source: Architecture docs (L1--L5) + CodeBase/BACKEND (Go, go-zero)

## Purpose

Software Requirements Document for the Memoket Backend -- the API gateway and microservice layer serving the Memoket mobile application. This document set covers all services, their functional requirements, data contracts, state models, error handling, NFRs, and observability.

## Document Map

| # | File | Description |
|---|------|-------------|
| 1 | [overview.md](overview.md) | Project objective, scope, non-scope, tech stack |
| 2 | [system-context.md](system-context.md) | System boundary, service decomposition, external dependencies |
| 3 | [nfr.md](nfr.md) | Quantified non-functional requirements (from config + code) |

## Module SRDs

| # | Module | File | Endpoints | DB Tables |
|---|--------|------|-----------|-----------|
| 01 | API Gateway & Middleware | [modules/01-api-gateway.md](modules/01-api-gateway.md) | (cross-cutting) | -- |
| 02 | User Service | [modules/02-user-service.md](modules/02-user-service.md) | ~26 | 8 |
| 03 | Audio Service | [modules/03-audio-service.md](modules/03-audio-service.md) | ~85 | 8 |
| 04 | Transcription Proxy | [modules/04-transcription-proxy.md](modules/04-transcription-proxy.md) | (subset of Audio) | -- |
| 05 | Export Service | [modules/05-export-service.md](modules/05-export-service.md) | 4 + worker | 1 |
| 06 | Notification Service | [modules/06-notification-service.md](modules/06-notification-service.md) | 3 | -- (Redis) |
| 07 | Integration Service | [modules/07-integration-service.md](modules/07-integration-service.md) | 8 | 2 |
| 08 | Task Service | [modules/08-task-service.md](modules/08-task-service.md) | ~9 | 3 |
| 09 | Template Service | [modules/09-template-service.md](modules/09-template-service.md) | ~8 | 3+ |
| 10 | MCP Service | [modules/10-mcp-service.md](modules/10-mcp-service.md) | 4 | 2 |

## Architecture References

- `Architecture/BACKEND/L1_system_overview.md` -- system overview
- `Architecture/BACKEND/L2_modules.md` -- module breakdown
- `Architecture/BACKEND/L3_data_flow.md` -- data flows and all REST endpoints
- `Architecture/BACKEND/L4_key_mechanisms.md` -- key mechanisms (JWT, AES, S3, SSE, export worker)
- `Architecture/BACKEND/L5_evaluation.md` -- architecture evaluation
- `Architecture/OVERALL/L3_cross_repo_flows.md` -- cross-repo API contracts
