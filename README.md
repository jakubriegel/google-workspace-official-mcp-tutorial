# Google Workspace MCP Server Tutorial for AI Agents on macOS (Gmail, Docs, Drive, Calendar) 🔧✨

This step-by-step tutorial shows how to run **gemini-cli-extensions/workspace** (official MCP tool from Google) as a **local MCP (Model Context Protocol) server** and connect it to an MCP-compatible AI client.
It is focused on common Google Workspace MCP and Gmail MCP setup queries on macOS, including OAuth login loops and localhost redirect issues.

---

## What you get ✅

After setup, your AI client can call Google Workspace tools such as:

- Google Docs (`docs.create`, `docs.getText`, …)
- Drive (`drive.search`, `drive.createFolder`, …)
- Gmail (`gmail.search`, draft/send helpers, …)
- Calendar (`calendar.getEvents`, …)
- Sheets/Slides/People/Chat (varies by tool set)

---

## Prerequisites 🧰

You need:

1) **Git**
2) **Node.js + npm** (recommended: Node 22+)
3) **An MCP-compatible AI client** installed
4) A **Google Cloud Project** with billing enabled
5) `gcloud` installed and authenticated

---

## Part A - Get the repo locally 📦

1) Clone the repo:

```bash
git clone https://github.com/gemini-cli-extensions/workspace.git
cd workspace
```

2. Install dependencies:

```bash
npm install
```

That’s it for the code installation.

---

## Part B - Set up authentication backend (self-hosted on GCP) 🔑

1. From the **repo root** (important!), run:

```bash
./scripts/setup-gcp.sh
```

> If you see `Invalid value for [--source]: Provided directory does not exist`
> it usually means you ran the script **outside** the repo root. `cd workspace` first.

2. When it finishes, it prints values for:

* `WORKSPACE_CLIENT_ID`
* `WORKSPACE_CLOUD_FUNCTION_URL`

3. Export those values in your shell (or put them into your MCP client config as env vars):

```bash
export WORKSPACE_CLIENT_ID="your-client-id"
export WORKSPACE_CLOUD_FUNCTION_URL="https://your-cloud-function-url"
```

---

## Part C - Force encrypted-file token storage 🔒

This avoids OS keychain issues and keeps everything local + predictable.

Set:

```bash
export GEMINI_CLI_WORKSPACE_FORCE_FILE_STORAGE=true
```

What this does:

* Tokens are stored in the repo root as:

  * `gemini-cli-workspace-token.json` (encrypted)
  * `.gemini-cli-workspace-master-key` (local key)

---

## Part D - Add the MCP server to AI Agent 🧩

Use the same server values in every agent:

* `command`: `node`
* `args`: `["scripts/start.js"]`
* `cwd`: absolute path to your cloned `workspace` folder
* env vars:
  * `GEMINI_CLI_WORKSPACE_FORCE_FILE_STORAGE = "true"`
  * `WORKSPACE_CLIENT_ID = "your-client-id"`
  * `WORKSPACE_CLOUD_FUNCTION_URL = "https://your-cloud-function-url"`

### Example 1: Codex (`~/.codex/config.toml`)

```toml
[mcp_servers.google_workspace]
command = "node"
args = ["scripts/start.js"]
cwd = "/ABSOLUTE/PATH/TO/workspace"
startup_timeout_sec = 60

[mcp_servers.google_workspace.env]
GEMINI_CLI_WORKSPACE_FORCE_FILE_STORAGE = "true"
WORKSPACE_CLIENT_ID = "your-client-id"
WORKSPACE_CLOUD_FUNCTION_URL = "https://your-cloud-function-url"
```

### Example 2: Cloude (`~/.cloude/mcp.json`)

```json
{
  "mcpServers": {
    "google_workspace": {
      "command": "node",
      "args": ["scripts/start.js"],
      "cwd": "/ABSOLUTE/PATH/TO/workspace",
      "env": {
        "GEMINI_CLI_WORKSPACE_FORCE_FILE_STORAGE": "true",
        "WORKSPACE_CLIENT_ID": "your-client-id",
        "WORKSPACE_CLOUD_FUNCTION_URL": "https://your-cloud-function-url"
      }
    }
  }
}
```

### Example 3: OpenClaw (`~/.openclaw/config.yaml`)

```yaml
mcp_servers:
  google_workspace:
    command: node
    args:
      - scripts/start.js
    cwd: /ABSOLUTE/PATH/TO/workspace
    env:
      GEMINI_CLI_WORKSPACE_FORCE_FILE_STORAGE: "true"
      WORKSPACE_CLIENT_ID: "your-client-id"
      WORKSPACE_CLOUD_FUNCTION_URL: "https://your-cloud-function-url"
```

Notes:

* `scripts/start.js` runs `npm install` (if needed) and then starts the MCP server.
* If your agent uses different key names, map the same values from the examples above.

---

## Part E - First run + login flow (Google Workspace/Gmail MCP) 🌐

### Normal flow (best case) ✅

1. Start your AI client.
2. Ask it to use any Workspace tool (for example: “Create a Google Doc titled Test”).
3. A browser window opens → sign into Google.
4. After success, the MCP server stores tokens (encrypted file).
5. Next tool call should *not* require login again.

### If Google Workspace or Gmail MCP keeps opening browser login 🐛

Usually means the client/server can’t reuse tokens (not stored, missing refresh token, wrong environment, or missing scopes).
If you hit serious troubles, contact me at `riegel@deltologic.com`.

Do these checks in order:

---

## Part F - Verify token persistence (fix repeated Gmail MCP login) 🔍

From the repo root:

```bash
npm run auth-utils -- status
```

You want to see:

* Access Token: ✅ Present
* Refresh Token: ✅ Present
* Status: ✅ Valid (or expired but refreshable)

If Refresh Token is missing, you’ll re-login every time.

---

## Part G - If Google Workspace MCP localhost redirect breaks (remote/container setup on MacOS) 🧯

Symptom:

* Login redirects to something like `http://localhost:<port>/oauth2callback...`
* But nothing is listening on that port (because the MCP server runs elsewhere).

### Fix: Force “manual” login mode 🧾

Set these env vars for the MCP server.
Codex TOML example (for Cloude and OpenClaw, set the same values in the configs from Part D):

```toml
[mcp_servers.google_workspace.env]
CI = "true"
GEMINI_CLI_WORKSPACE_FORCE_FILE_STORAGE = "true"
```

Now, instead of trying to complete a localhost callback, the server prints a URL.

1. Run a tool again.
2. Open the printed URL in a browser and log in.
3. You’ll get a JSON blob containing tokens.

### Import that JSON into encrypted-file storage ✅

1. Save the JSON to a local file, e.g. `creds.json` (DO NOT COMMIT IT).

2. Run:

```bash
# From repo root:
npm run build:auth-utils -w workspace-server

node -e 'const {OAuthCredentialStorage}=require("./workspace-server/dist/auth-utils.js"); const creds=require("./creds.json"); OAuthCredentialStorage.saveCredentials(creds).then(()=>console.log("✅ saved")).catch(e=>{console.error(e); process.exit(1);});'
```

3. Confirm:

```bash
npm run auth-utils -- status
```

Now your AI client should stop re-opening login pages.

---


## Part H - Quick sanity test prompts 🧪

Try these after setup:

* “List my next 5 calendar events.”
* “Search Gmail for unread emails from the last 7 days.”
* “Find a Google Drive file named ‘README’.”

If these work without repeated login, you’re done ✅🎉

---

## FAQ - Google Workspace/Gmail MCP quick answers

**How do I set up Google Workspace MCP on macOS?**
Follow Part A through Part D.

**Why does Gmail MCP ask me to log in every time?**
Usually the refresh token is missing or invalid. Check Part F, then Part G.

**How do I fix localhost OAuth callback issues for Google Workspace MCP?**
Use manual login mode (`CI=true`) from Part G.

**Which Google tools does this MCP server support?**
Gmail, Google Docs, Drive, Calendar, Sheets, Slides, People, and Chat (depends on enabled tools).

---

We also build advanced AI agents and automations.
Contact me at `riegel@deltologic.com`.
