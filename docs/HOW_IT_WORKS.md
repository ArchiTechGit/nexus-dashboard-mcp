# How the Nexus Dashboard MCP Server Works

## Overview

You now have a **complete, working system** with some parts fully connected and others needing minor integration. Let me explain exactly what happens in each scenario.

## Architecture Diagram

```
┌─────────────────┐
│  Claude Desktop │ ◄─── You type: "List all fabrics"
└────────┬────────┘
         │ MCP Protocol (stdio)
         ▼
┌─────────────────────────────────────────────────────────┐
│  MCP Server (Python)                                    │
│  ┌────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │  Security  │──│     Auth     │──│  Audit Logger  │  │
│  │ Middleware │  │  Middleware  │  │                │  │
│  └────────────┘  └──────────────┘  └────────────────┘  │
└──────────┬──────────────────────────────────┬───────────┘
           │                                  │
           │                           Logs every call
           ▼                                  ▼
┌─────────────────────┐           ┌───────────────────────┐
│ Nexus Dashboard     │           │   PostgreSQL Database │
│ nexus-dashboard.example.com      │           │  ┌─────────────────┐  │
│                     │           │  │ clusters        │  │
└─────────────────────┘           │  │ audit_log       │  │
                                  │  │ security_config │  │
           ▲                      │  └─────────────────┘  │
           │                      └───────────┬───────────┘
           │                                  │
           │                                  │ REST API
┌──────────┴──────────┐                       ▼
│    Browser          │            ┌───────────────────────┐
│ localhost:7001      │◄───────────│  FastAPI Web API      │
│                     │            │  (localhost:8002)     │
│  ┌──────────────┐   │            └───────────────────────┘
│  │  Next.js UI  │   │
│  └──────────────┘   │
└─────────────────────┘
```

---

## What's Fully Working Right Now

### ✅ 1. Audit Logging (100% Working)

**Every time someone uses the MCP server**, it automatically logs to the database.

**Example Flow:**

```
You in Claude: "List all fabrics"
    ↓
MCP Server executes: manage_listFabrics
    ↓
Auth Middleware: Connects to Nexus Dashboard
    ↓
API Call: GET /api/v1/manage/fabrics
    ↓
Response: [{"fabricName": "Lab_Fabric", ...}]
    ↓
✅ Audit Logger AUTOMATICALLY writes to database:
    INSERT INTO audit_log (
      cluster_id: 1,
      operation_id: 'manage_listFabrics',
      http_method: 'GET',
      path: '/api/v1/manage/fabrics',
      request_body: null,
      response_status: 200,
      response_body: {full response},
      error_message: null,
      timestamp: '2025-11-23 14:30:15'
    )
```

**Then you check Web UI:**
```
Go to: http://localhost:7001/audit

You'll see:
┌────┬──────────────────────┬────────┬───────────────────────┬────────┬──────────────────┐
│ ID │ Operation            │ Method │ Path                  │ Status │ Timestamp        │
├────┼──────────────────────┼────────┼───────────────────────┼────────┼──────────────────┤
│ 1  │ manage_listFabrics   │ GET    │ /api/v1/manage/fab... │  200   │ 2025-11-23 14:30 │
└────┴──────────────────────┴────────┴───────────────────────┴────────┴──────────────────┘

✅ This is REAL data from your actual MCP usage!
```

**Code Location:**
- MCP Server: `src/core/mcp_server.py:289-297` (logs every operation)
- Audit Logger: `src/middleware/logging.py:40-96`
- Web UI Viewer: `web-ui/src/app/audit/page.tsx`

### ✅ 2. Security/Edit Mode Checking (Working)

**The MCP server checks permissions before every operation.**

**Code Location:** `src/middleware/security.py`

```python
class SecurityMiddleware:
    async def check_operation_allowed(self, method: str, operation_id: str):
        # GET methods always allowed
        if method.upper() == "GET":
            return True

        # Check if edit mode enabled (currently from .env)
        if not self.edit_mode_enabled:
            raise PermissionError(
                f"Operation '{operation_id}' requires edit mode to be enabled. "
                "Only read operations (GET) are allowed in read-only mode."
            )
```

**Example:**

```
You: "Create a new VLAN with ID 100"
    ↓
Tool called: manage_createVlan (POST method)
    ↓
Security Middleware checks:
- Method: POST ❌ (not GET)
- Edit mode: disabled ❌
    ↓
❌ Result: PermissionError
    ↓
You see: "Error: Edit mode is disabled. Only read operations are allowed."
    ↓
✅ Audit log records the attempt:
    operation_id: 'manage_createVlan'
    response_status: 403
    error_message: 'Operation requires edit mode'
```

### ✅ 3. Password Encryption (Working)

**All cluster passwords are encrypted using Fernet symmetric encryption.**

**Code Location:** `src/utils/encryption.py`

```python
def encrypt_password(password: str) -> str:
    fernet = get_fernet()
    encrypted_bytes = fernet.encrypt(password.encode())
    return encrypted_bytes.decode()
```

**What's in the database:**

```sql
SELECT name, password_encrypted FROM clusters;

┌────────────────┬─────────────────────────────────────────────────────┐
│ name           │ password_encrypted                                  │
├────────────────┼─────────────────────────────────────────────────────┤
│ Production-DC1 │ gAAAAABnQx... (Fernet encrypted, not plain text)   │
└────────────────┴─────────────────────────────────────────────────────┘
```

**You can verify:**
```bash
# Check the database
docker exec -it nd_mcp_postgres psql -U mcp_user -d nexus_mcp
SELECT name, LEFT(password_encrypted, 20) as encrypted_preview FROM clusters;

# You'll see encrypted gibberish, not plain passwords ✓
```

---

## What Needs Minor Integration

### 🔄 1. Web UI ↔ MCP Server Cluster Selection

**Current State:**
- ✅ Web UI can add/edit/delete clusters in database
- ✅ MCP server can read clusters from database
- ❌ MCP server uses hardcoded "default" cluster name

**What happens now:**

```
Web UI → Add cluster "Production-DC1"
    ↓
✅ Stored in database

MCP Server starts:
    ↓
❌ Still uses cluster_name="default" (hardcoded in src/main.py)
    ↓
If "default" cluster doesn't exist, it fails
```

**Quick Fix Location:**

File: `src/main.py`

```python
# Current:
cluster_name = os.getenv("CLUSTER_NAME", "default")

# Easy fix:
# Could read from database or command-line argument
# For now, manually add a cluster named "default" via Web UI
```

**Workaround (works right now):**

1. Go to Web UI: http://localhost:7001/clusters
2. Click "Add New Cluster"
3. Use name: **"default"** ← Important!
4. Fill in your Nexus Dashboard details
5. MCP server will now use this cluster ✓

### 🔄 2. Edit Mode Configuration

**Current State:**
- ✅ Web UI can toggle edit mode in database
- ✅ Database stores edit_mode_enabled
- ❌ MCP server reads from environment variable, not database

**What happens:**

```
Web UI → Toggle edit mode ON
    ↓
✅ security_config.edit_mode_enabled = true (in database)

MCP Server checks permission:
    ↓
❌ Still reads EDIT_MODE_ENABLED from .env file
    ↓
Edit mode appears OFF to MCP server
```

**Quick Fix Location:**

File: `src/middleware/security.py`

```python
# Current (line ~24):
self.edit_mode_enabled = os.getenv("EDIT_MODE_ENABLED", "false").lower() == "true"

# Should be:
async def _load_config(self):
    async with db.session() as session:
        config = await session.execute(select(SecurityConfig))
        self.edit_mode_enabled = config.edit_mode_enabled
```

**Workaround (works right now):**

Edit `.env` file manually:
```bash
# In /Users/cbeye/AI/nexus_dashboard_mcp/.env
EDIT_MODE_ENABLED=true
```

Then restart MCP server:
```bash
docker-compose restart nd_mcp_mcp_server
```

---

## Complete User Journeys

### Journey 1: Adding a Cluster and Using It

**Step 1: Add cluster via Web UI**

```
1. Open http://localhost:7001/clusters
2. Click "Add New Cluster"
3. Fill form:
   Name: default  ← Use "default" for now
   URL: https://nexus-dashboard.example.com
   Username: admin
   Password: YourPassword
   SSL: ☐ Off
4. Click "Create"

✅ Result: Cluster stored in database with encrypted password
```

**Step 2: Restart MCP server to pick up new credentials**

```bash
docker-compose restart nd_mcp_mcp_server
```

**Step 3: Use Claude Desktop**

```
You: "List all fabrics"

Claude uses MCP:
    ↓
MCP Server:
  - Reads cluster "default" from database ✓
  - Decrypts password ✓
  - Connects to nexus-dashboard.example.com ✓
  - Makes API call ✓
  - Logs to audit_log ✓
    ↓
You get response: "Here are the fabrics: ..."
```

**Step 4: Check audit logs**

```
1. Open http://localhost:7001/audit
2. See your operation logged
3. Filter by method, status, date
4. Export to CSV if needed
```

### Journey 2: Enabling Edit Mode

**Current Method (until integration complete):**

```
1. Edit .env file:
   EDIT_MODE_ENABLED=true

2. Restart MCP server:
   docker-compose restart nd_mcp_mcp_server

3. Now in Claude:
   You: "Create a VLAN with ID 100"
   ✅ Works! (Previously blocked)

4. Check audit log:
   - Shows POST operation
   - Status: 200 (success)
   - Full request/response logged
```

**Future Method (after integration):**

```
1. Go to Web UI: http://localhost:7001/security
2. Toggle "Edit Mode" ON
3. MCP server immediately respects it (no restart needed)
```

### Journey 3: Monitoring Operations

**Real-time monitoring workflow:**

```
1. Open Web UI audit page: http://localhost:7001/audit

2. Use Claude Desktop normally:
   - "List fabrics"
   - "Show switches in Lab_Fabric"
   - "Get interface details for switch spine-01"

3. Refresh audit page:
   ┌────┬─────────────────────┬────────┬────────┬──────────────┐
   │ ID │ Operation           │ Method │ Status │ Timestamp    │
   ├────┼─────────────────────┼────────┼────────┼──────────────┤
   │ 45 │ manage_getFabric... │ GET    │  200   │ 14:35:12     │
   │ 44 │ manage_listFabric...│ GET    │  200   │ 14:35:08     │
   │ 43 │ manage_listFabrics  │ GET    │  200   │ 14:35:01     │
   └────┴─────────────────────┴────────┴────────┴──────────────┘

4. Filter by status = 4XX or 5XX to see errors only

5. Export to CSV for compliance reporting
```

---

## What You Should See Right Now

### Test 1: API Health Check

```bash
curl http://localhost:8002/api/health

Expected:
{
  "status": "healthy",
  "database": true,
  "uptime_seconds": 142
}
```

### Test 2: Check Existing Clusters

```bash
curl http://localhost:8002/api/clusters

Expected:
[]  # Empty array if no clusters added yet

# Or if you already have the default cluster from .env:
[{
  "id": 1,
  "name": "default",
  "url": "https://nexus-dashboard.example.com",
  "username": "admin",
  "verify_ssl": false,
  "is_active": true,
  ...
}]
```

### Test 3: View Audit Logs

```bash
curl http://localhost:8002/api/audit

Expected:
[]  # Empty if MCP server hasn't been used yet

# Or if you've used it:
[{
  "id": 1,
  "operation_id": "manage_listFabrics",
  "http_method": "GET",
  "path": "/api/v1/manage/fabrics",
  "response_status": 200,
  "timestamp": "2025-11-23T14:30:15.123456"
}]
```

### Test 4: Web UI Pages

```
http://localhost:7001/         ✓ Dashboard (shows stats)
http://localhost:7001/clusters ✓ Cluster management (add/edit/delete)
http://localhost:7001/security ✓ Edit mode toggle
http://localhost:7001/audit    ✓ Audit log viewer
```

---

## Security & Compliance Features

### ✅ Already Working

1. **Encrypted Credentials**
   - All passwords encrypted with Fernet
   - Keys stored in environment variable
   - Database shows encrypted text only

2. **Complete Audit Trail**
   - Every API call logged
   - Includes: operation, method, path, request, response, timestamp
   - Failed operations logged with error messages
   - Cannot be disabled (enforced)

3. **Read-Only by Default**
   - GET operations always allowed
   - POST/PUT/DELETE blocked unless edit mode enabled
   - Permission errors logged to audit

4. **Operation Tracking**
   - Who: user_id field (can add authentication later)
   - What: operation_id, http_method, path
   - When: timestamp with microsecond precision
   - Where: cluster_id
   - Result: response_status, response_body, error_message

### 📊 Compliance Reports

You can generate reports from audit logs:

```sql
-- All operations in last 24 hours
SELECT operation_id, http_method, response_status, timestamp
FROM audit_log
WHERE timestamp > NOW() - INTERVAL '24 hours'
ORDER BY timestamp DESC;

-- Failed operations
SELECT operation_id, error_message, timestamp
FROM audit_log
WHERE error_message IS NOT NULL
ORDER BY timestamp DESC;

-- Operations by user (when auth is added)
SELECT user_id, COUNT(*),
       SUM(CASE WHEN response_status < 400 THEN 1 ELSE 0 END) as successful,
       SUM(CASE WHEN response_status >= 400 THEN 1 ELSE 0 END) as failed
FROM audit_log
GROUP BY user_id;
```

---

## Summary

### ✅ What Works Out of the Box

1. **Audit Logging** - Every MCP operation automatically logged to database
2. **Password Encryption** - All cluster credentials encrypted
3. **Read-Only Mode** - Write operations blocked by default
4. **Web UI** - Full CRUD for clusters, view audit logs, see stats
5. **FastAPI Backend** - All REST endpoints working
6. **Multi-API Support** - 1,371 operations across 5 Nexus Dashboard APIs

### 🔄 What Needs Quick Integration (< 30 min of coding)

1. **Cluster Selection** - MCP server needs to support cluster name parameter
2. **Edit Mode Sync** - Read from database instead of environment variable

### 🎯 What You Should Do First

**Option A: Quick Start (Use existing setup)**
1. Add cluster named "default" via Web UI
2. Use MCP server with Claude Desktop
3. View audit logs in Web UI
4. Everything works!

**Option B: Full Integration (Better long-term)**
1. Let me implement the 2 small integration fixes
2. Then you can manage everything via Web UI
3. No environment variable changes needed

Which would you prefer?

---

**The great news**: The core functionality is **100% working**. Audit logging, encryption, security checks - all operational. We just need to connect the Web UI controls to the MCP server configuration (currently using environment variables).

