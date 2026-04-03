# Module 10 -- MCP Service

> Generated: 2026-04-02

## 1. Overview

### Objective
Provide Model Context Protocol (MCP) integration capabilities, allowing the Memoket app to connect with external tool services via OAuth. The MCP service manages available MCP servers, user OAuth authorization for those servers, and exposes an MCP SSE server endpoint for tool execution.

### Scope
- MCP server listing (available tools/integrations)
- MCP server enable/disable toggle per user
- OAuth authorization flow for MCP servers (auth URL generation, callback, token storage)
- MCP SSE server endpoint for tool execution
- MCP server metadata and auth data management

### Non-Scope
- Tool execution logic (handled by external MCP servers)
- AI processing (AI service)
- Other integration types (Notion/Slack handled by integration service)

## 2. Definitions

| Term | Definition |
|------|-----------|
| MCP | Model Context Protocol -- standardized protocol for LLM tool integrations |
| MCP Server | External service that exposes tools via the MCP protocol |
| mcp_server | Database record describing an available MCP server (name, URL, protocol) |
| mcp_server_auth | Database record storing user's OAuth tokens for an MCP server |
| DCR | Dynamic Client Registration -- OAuth capability; if unsupported, pre-configured client_id/secret used |
| protocol | Server communication type: "http" (REST-based) or "app" (native) |

## 3. System Boundary

| This service handles | Other services handle |
|--------------------|---------------------|
| MCP server metadata management | External MCP server tool execution |
| User OAuth flow for MCP servers | AI chat integration with MCP tools |
| Auth token storage | Google Calendar OAuth (user service) |
| MCP SSE server endpoint | Other integration auth (integration service) |

### gRPC Dependencies
- **mcp-api** calls: mcp-rpc
- **mcp-rpc** is standalone (no downstream RPC dependencies)

## 4. Scenarios

### SC-MCP-01: List Available MCP Apps
1. Client sends `GET /api/v1/mcp`
2. mcp-api -> mcp-rpc: `GetMcpServerListWithAuth` with userId
3. Returns list of MCP servers with their auth status per user
4. Client displays enabled/disabled state and authorization status

### SC-MCP-02: Toggle MCP App
1. Client sends `POST /api/v1/mcp/:id` to enable/disable an MCP app
2. mcp-api -> mcp-rpc: `UpdateMcpServerStatus` with server_id and desired status
3. Returns success

### SC-MCP-03: OAuth Authorization for MCP Server
1. Client sends `GET /api/v1/mcp/:id/auth_url`
2. mcp-api -> mcp-rpc: `GetMcpServerById` to get server config (base_url, client_id, client_secret, oauth_scope)
3. mcp-api uses `common/mcp/` OAuth client to construct authorization URL
4. If server supports DCR: performs Dynamic Client Registration first
5. Returns `{auth_url}`
6. User completes OAuth in browser
7. Provider redirects to `GET /api/v1/mcp/auth_callback?code=XXX&state=...`
8. mcp-api exchanges code for tokens
9. mcp-api -> mcp-rpc: `SaveMcpServerAuth` stores tokens
10. Returns success/redirect

### SC-MCP-04: MCP SSE Server Endpoint
1. External MCP client connects to MCP SSE server endpoint
2. Server maintains SSE connection for tool execution requests
3. Tool calls are dispatched to the appropriate MCP server using stored auth tokens

## 5. Functional Requirements

| ID | Priority | Requirement | Verification |
|----|----------|-------------|--------------|
| FR-MCP-01 | MUST | MCP server list MUST return all configured servers with per-user auth status | Test: list returns servers with auth_status field |
| FR-MCP-02 | MUST | Toggle endpoint MUST update server status (enable/disable) for the user | Test: status toggles correctly |
| FR-MCP-03 | MUST | OAuth URL generation MUST use server-specific client_id, client_secret, and oauth_scope | Test: generated URL contains correct parameters |
| FR-MCP-04 | MUST | OAuth callback MUST exchange code for tokens and store via SaveMcpServerAuth | Test: tokens stored after callback |
| FR-MCP-05 | MUST | Token storage MUST include access_token, refresh_token, client_id, client_secret, expires_at | Test: all fields persisted |
| FR-MCP-06 | SHOULD | Server config SHOULD support per-server oauth_scope (falling back to global scope if empty) | Test: custom scope used when configured |
| FR-MCP-07 | SHOULD | Dynamic Client Registration SHOULD be attempted if server supports DCR | Test: DCR flow executed for DCR-capable servers |
| FR-MCP-08 | MUST | GetMcpServerWithAuthByUserId MUST return server details with auth tokens for tool execution | Test: tokens returned for authorized servers |
| FR-MCP-09 | MUST | Auth callback endpoint MUST be publicly accessible (no JWT auth) | Test: callback without auth succeeds |

## 6. State Model

### MCP Server Status (per user)
```
0 (disabled) <-> 1 (enabled)
```

### MCP Auth Status (per user per server)
```
0 (not authorized) -> 1 (authorized) -> 0 (expired/revoked)
```

## 7. Data Contract

### REST API Endpoints

| Method | Path | Auth | Request | Response |
|--------|------|------|---------|----------|
| GET | `/api/v1/mcp` | Yes | -- | `{list: [{id, name, protocol, base_url, description, status, auth_status}]}` |
| POST | `/api/v1/mcp/:id` | Yes | `{status: 0\|1}` | `{success}` |
| GET | `/api/v1/mcp/:id/auth_url` | Yes | -- | `{auth_url}` |
| GET | `/api/v1/mcp/auth_callback` | No | `?code=XXX&state=...` | Redirect/success |

### gRPC Service: `mcp` (7 RPCs)

```protobuf
service mcp {
  rpc GetMcpServerList(GetMcpServerListReq) returns(GetMcpServerListResp);
  rpc UpdateMcpServerStatus(UpdateMcpServerStatusReq) returns(UpdateMcpServerStatusResp);
  rpc GetMcpServerById(GetMcpServerByIdReq) returns(GetMcpServerByIdResp);
  rpc SaveMcpServerAuth(SaveMcpServerAuthReq) returns(SaveMcpServerAuthResp);
  rpc GetMcpServerAuth(GetMcpServerAuthReq) returns(GetMcpServerAuthResp);
  rpc GetMcpServerWithAuthByUserId(GetMcpServerWithAuthByUserIdReq) returns(GetMcpServerWithAuthByUserIdResp);
  rpc GetMcpServerListWithAuth(GetMcpServerListWithAuthReq) returns(GetMcpServerListWithAuthResp);
}
```

### Key Proto Messages

**McpServerInfo** (list response):
```protobuf
message McpServerInfo {
  int64 id = 1;
  string name = 2;
  string protocol = 3;      // "http" or "app"
  string base_url = 4;
  string description = 5;
  int64 status = 6;
  int64 auth_status = 7;    // 0=not authorized, 1=authorized
}
```

**McpServerWithAuth** (for tool execution):
```protobuf
message McpServerWithAuth {
  int64 server_id = 1;
  string server_name = 2;
  string base_url = 3;
  int64 auth_id = 4;
  string auth_type = 5;      // "oauth" or "api_key"
  string access_token = 6;
  string refresh_token = 7;
  string client_id = 8;
  string client_secret = 9;
  int64 expires_at = 10;
  int64 auth_status = 11;
  string protocol = 12;
}
```

### Database Tables

| Table | Key Columns |
|-------|-------------|
| `mcp_server` | id, name, protocol, base_url, description, status, client_id, client_secret, oauth_scope |
| `mcp_server_auth` | id, user_id, server_id, auth_type, access_token, refresh_token, client_id, client_secret, expires_at, status |

## 8. Error Handling

| Error | Code | Handling |
|-------|------|---------|
| MCP server not found | 404 | Return "MCP server not found" |
| OAuth code exchange failure | 500 | Log error; return "authorization failed" |
| DCR failure | 500 | Fall back to pre-configured client_id/secret if available |
| Token expired | 401 | Attempt token refresh; if failed, return "re-authorization required" |
| External MCP server unavailable | 502 | Log error; return "MCP server unavailable" |

## 9. NFR (Quantified)

| Metric | Value | Source |
|--------|-------|--------|
| gRPC client timeout (mcp-rpc from audio-api) | 5,000 ms | `audio-api: McpRpc.Timeout: 5000` |
| OAuth client | `common/mcp/` | MCP-specific OAuth implementation |
| Google OAuth client | `common/third_party/google/` | Shared with user/task services |
| Auth types supported | oauth, api_key | `auth_type` field |
| Protocols supported | http, app | `protocol` field |

## 10. Observability

| Signal | Implementation |
|--------|---------------|
| MCP server list access | Logged with userId |
| OAuth flow events | Provider, server_id, userId, success/failure logged |
| Toggle operations | Status changes logged |
| Token refresh | Refresh attempts and outcomes logged |
| Trace propagation | X-Trace-ID through all RPC calls |
