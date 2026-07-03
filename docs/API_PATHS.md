# Nexus Dashboard API Path Reference

## API Base Paths

All five APIs are loaded and enabled. Each API is defined by an OpenAPI spec in
`openapi_specs/` and registered in `src/core/api_registry.py`, which supplies the
base path used to route requests.

| API | Spec file | Version | Base path | Server URL | Operations |
|-----|-----------|---------|-----------|------------|-----------:|
| Manage | `manage.json` | 1.1.411 | `/api/v1/manage` | `https://{cluster}/api/v1/manage` | 497 |
| Analyze | `analyze.json` | 1.1.209 | `/api/v1/analyze` | `https://{cluster}/api/v1/analyze` | 316 |
| Infrastructure | `infra.json` | 1.1.136 | `/api/v1/infra` | `https://{cluster}/api/v1/infra` | 280 |
| OneManage | `oneManage.json` | 1.1.218 | `/api/v1/oneManage` | `https://{cluster}/api/v1/oneManage` | 132 |
| Orchestrator | `orchestration.json` | 5.2.1 | `/mso` | `https://{cluster}/mso` | 146 |
| **Total** | | | | | **1,371** |

> **Note:** The Orchestrator (Multi-Site Orchestrator) API is served under `/mso`,
> **not** under `/api/v1/...` like the other four.

**Example Manage endpoints**:
- List Fabrics: `GET /api/v1/manage/fabrics`
- Get Fabric: `GET /api/v1/manage/fabrics/{fabricId}`
- List Anomalies: `GET /api/v1/manage/anomalyRules/complianceRules`

## OpenAPI Specification Structure

### How Paths are Defined

In the OpenAPI spec (`manage.json`):

```json
{
  "openapi": "3.0.3",
  "servers": [
    {
      "url": "https://{cluster}/api/v1/manage",
      "variables": {
        "cluster": {
          "default": "example.com"
        }
      }
    }
  ],
  "paths": {
    "/fabrics": {
      "get": {
        "operationId": "listFabrics",
        ...
      }
    }
  }
}
```

**Key Points**:
- The `servers[0].url` contains the full base path: `https://{cluster}/api/v1/manage`
- The `paths` object contains relative paths like `/fabrics`
- Full URL is: `servers[0].url + paths.key` = `https://{cluster}/api/v1/manage/fabrics`

## Implementation in MCP Server

### Path Construction Flow

1. **API Loader** (`src/core/api_loader.py`):
   - Reads OpenAPI spec
   - Extracts paths from `paths` object (e.g., `/fabrics`)
   - Does NOT include the base path from `servers[0].url`

2. **MCP Server** (`src/core/mcp_server.py`):
   - Gets operation with path `/fabrics`
   - Passes it to AuthMiddleware

3. **Auth Middleware** (`src/middleware/auth.py`):
   - **CRITICAL**: Prepends `/api/v1/manage` to the path
   - Code:
     ```python
     if not path.startswith("/api/"):
         path = f"/api/v1/manage{path}"
     ```
   - Final path: `/api/v1/manage/fabrics`

4. **Nexus API Client** (`src/services/nexus_api.py`):
   - Receives full path: `/api/v1/manage/fabrics`
   - Joins with base_url: `https://nexus-dashboard.example.com`
   - Final URL: `https://nexus-dashboard.example.com/api/v1/manage/fabrics`

## Authentication Endpoints

### Login
**Endpoint**: `POST /login`
**Full URL**: `https://{cluster}/login`

**Note**: Login endpoint is NOT under `/api/v1/manage`. It's a top-level endpoint.

**Request Body**:
```json
{
  "username": "admin",
  "password": "password"
}
```

**Response**:
- Sets session cookies
- May return token in response body

## Common Issues and Solutions

### Issue: 404 Not Found

**Symptom**: API returns 404 when calling endpoints

**Possible Causes**:
1. Missing base path (`/api/v1/manage`)
2. Wrong API version
3. Service not running
4. Incorrect cluster URL

**Debugging Steps**:
```bash
# Check what URL is being called
docker-compose logs mcp-server | grep "API request"

# Test endpoint manually
curl -k https://nexus-dashboard.example.com/api/v1/manage/fabrics

# Verify cluster is accessible
curl -k https://nexus-dashboard.example.com
```

### Issue: 401 Unauthorized

**Symptom**: API returns 401

**Possible Causes**:
1. Authentication failed
2. Session expired
3. Invalid credentials

**Solution**: Check authentication in AuthMiddleware

### Issue: Wrong Base Path

**Symptom**: Calls going to wrong API (e.g., Analyze instead of Manage)

**Solution**: Verify the base path logic in `src/middleware/auth.py`

## Testing API Paths

### Manual Testing

```bash
# 1. Get authentication token
curl -k -X POST https://nexus-dashboard.example.com/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"Davinci!!02"}' \
  -c cookies.txt

# 2. Test Manage API endpoint
curl -k -X GET https://nexus-dashboard.example.com/api/v1/manage/fabrics \
  -b cookies.txt

# 3. Test with specific fabric
curl -k -X GET https://nexus-dashboard.example.com/api/v1/manage/fabrics/{fabricId} \
  -b cookies.txt
```

### Automated Testing

See `src/services/nexus_api.py` for the `NexusAPIClient` class that handles:
- Authentication with cookies
- Automatic retry on 401
- Session management
- Path construction

## Multi-API Support

Multi-API routing is implemented. Each operation is tagged with its `api_name`
when tools are generated, and `AuthMiddleware.execute_request()` resolves the
base path from `APIRegistry` at request time:

```python
# In src/middleware/auth.py
base_path = APIRegistry.get_base_path_for_api(api_name) or "/api/v1/manage"
if not path.startswith(base_path):
    path = f"{base_path}{path}"
```

Base paths come from `src/core/api_registry.py`:

```python
"manage":        "/api/v1/manage",
"analyze":       "/api/v1/analyze",
"infra":         "/api/v1/infra",
"onemanage":     "/api/v1/oneManage",
"orchestration": "/mso",
```

> **Why the guard checks the base path, not a fixed `/api/` prefix:** The
> Orchestrator API is served under `/mso`, but its spec paths themselves already
> start with `/api/v1/...` (e.g. `/api/v1/platform/version`). An older
> `if not path.startswith("/api/")` guard would have skipped prepending `/mso`
> and routed Orchestrator calls to the wrong endpoint. Checking against each
> API's own `base_path` prepends correctly for all five APIs and is idempotent:
>
> - Manage `/fabrics` → `/api/v1/manage/fabrics`
> - OneManage `/manage/fabrics` → `/api/v1/oneManage/manage/fabrics`
> - Orchestrator `/api/v1/platform/version` → `/mso/api/v1/platform/version`

## References

- Nexus Dashboard API Documentation: https://developer.cisco.com/docs/nexus-dashboard/
- OpenAPI 3.0 Specification: https://swagger.io/specification/
- OpenAPI specs: `openapi_specs/manage.json`, `analyze.json`, `infra.json`, `oneManage.json`, `orchestration.json`

---

**Last Updated**: July 3, 2026
**Status**: Active documentation — all five APIs enabled
