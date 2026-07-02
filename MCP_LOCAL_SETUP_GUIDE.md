# MCP Local Setup Guide

## This guide sets up Codex MCP servers locally on Windows, without Docker or Rancher Desktop.

For Codex MCP servers we can avoid running Docker through Rancher Desktop and use a local setup instead. Local MCP is usually faster to start, consumes less memory, and has fewer moving parts because it does not need the Rancher Desktop VM or containers running in the background.

Recommended local approach:

- Jira and Wiki/Confluence: run `mcp-atlassian` directly using `uvx`
- GitHub Enterprise: run the local `github-mcp-server.exe`
- Keep Docker/Rancher Desktop only as a fallback if local setup fails

This setup still uses the same PATs and same Jira/Wiki/GitHub URLs. The main change is that Codex launches the MCP servers as local processes instead of Docker containers.

## What This Replaces

Old approach:

```text
Codex -> Docker CLI -> Rancher Desktop VM -> MCP container -> Jira/Wiki/GitHub
```

New approach:

```text
Codex -> local MCP process -> Jira/Wiki/GitHub
```

## Benefits

- Faster MCP server startup
- Lower memory usage
- No need to keep Rancher Desktop running for MCP
- Easier troubleshooting because the MCP server runs directly on Windows
- Simple token rotation by editing `config.toml`

## Prerequisites

- Windows machine with Codex installed
- VPN or internal network access, if required for Cerner services
- PATs for:
  - Jira: `https://jira2.cerner.com`
  - Wiki/Confluence: `https://wiki.cerner.com`
  - GitHub Enterprise: `https://github.cerner.com`

## Step 1: Install `uvx` for Jira and Wiki MCP

Open PowerShell and run:

```powershell
winget install --id astral-sh.uv -e
```

Close PowerShell and open it again, then verify:

```powershell
uvx --version
```

Test the Atlassian MCP server:

```powershell
uvx mcp-atlassian --help
```

Note: the first run may take some time because `uvx` downloads the `mcp-atlassian` package.

## Step 2: Install Git and Go for GitHub MCP

Open PowerShell and run:

```powershell
winget install --id Git.Git -e
winget install --id GoLang.Go -e
```

Close PowerShell and open it again, then verify:

```powershell
git --version
go version
```

## Step 3: Build the GitHub MCP Server

Run:

```powershell
New-Item -ItemType Directory -Force C:\Tools\github-mcp-server-bin
cd C:\Tools
git clone https://github.com/github/github-mcp-server.git github-mcp-server-src
cd C:\Tools\github-mcp-server-src
go build -o C:\Tools\github-mcp-server-bin\github-mcp-server.exe .\cmd\github-mcp-server
```

Verify:

```powershell
C:\Tools\github-mcp-server-bin\github-mcp-server.exe --help
```

## Step 4: Configure Codex MCP

Open the Codex config file:

```powershell
notepad "$env:USERPROFILE\.codex\config.toml"
```

Add or update the MCP configuration below.

Important:

- Replace only the `PASTE_..._PAT_HERE` values.
- Do not paste URLs in Markdown format like `[https://...](https://...)`.
- PATs stored in TOML are plaintext. Keep this file private to your user account.

```toml
# =========================
# Jira - local MCP
# =========================

[mcp_servers.jira2]
command = "uvx"
args = ["mcp-atlassian"]
startup_timeout_sec = 300
tool_timeout_sec = 120
enabled = true

[mcp_servers.jira2.env]
JIRA_URL = "https://jira2.cerner.com"
JIRA_PERSONAL_TOKEN = "PASTE_JIRA_PAT_HERE"
JIRA_SSL_VERIFY = "false"


# =========================
# Wiki / Confluence - local MCP
# =========================

[mcp_servers.confluence]
command = "uvx"
args = ["mcp-atlassian"]
startup_timeout_sec = 300
tool_timeout_sec = 120
enabled = true

[mcp_servers.confluence.env]
CONFLUENCE_URL = "https://wiki.cerner.com"
CONFLUENCE_PERSONAL_TOKEN = "PASTE_WIKI_PAT_HERE"
CONFLUENCE_SSL_VERIFY = "false"
CONFLUENCE_SPACES_FILTER = "CCBHIP"
READ_ONLY_MODE = "true"


# =========================
# GitHub Enterprise - local MCP
# =========================

[mcp_servers.github]
command = "C:\\Tools\\github-mcp-server-bin\\github-mcp-server.exe"
args = ["stdio"]
startup_timeout_sec = 60
tool_timeout_sec = 120
enabled = true

[mcp_servers.github.env]
GITHUB_HOST = "https://github.cerner.com"
GITHUB_PERSONAL_ACCESS_TOKEN = "PASTE_GITHUB_PAT_HERE"
```

## Step 5: Restart Codex

Fully close and reopen Codex after editing `config.toml`.

If using Codex CLI or another surface that supports it, run:

```text
/mcp
```

Confirm that Jira, Confluence, and GitHub MCP servers are listed.

## Token Rotation

Because the PATs are stored in `config.toml`, token rotation is simple:

1. Generate a new PAT.
2. Open `C:\Users\<your-user>\.codex\config.toml`.
3. Replace the old token value.
4. Restart Codex.

Example:

```toml
JIRA_PERSONAL_TOKEN = "PASTE_NEW_JIRA_PAT_HERE"
```

## Troubleshooting

### `uvx` is not recognized

Close and reopen PowerShell. If it still fails, restart Windows or reinstall `uv`.

### `uvx mcp-atlassian --help` fails

Common causes:

- No internet access for the first package download
- Corporate proxy issue
- SSL inspection or certificate issue
- VPN not connected

### GitHub MCP exe is not found

Check that this file exists:

```text
C:\Tools\github-mcp-server-bin\github-mcp-server.exe
```

If the path is different, update the `command` value in `config.toml`.

### Authentication fails

Check:

- PAT is valid and not expired
- PAT has the required permissions
- URL is correct
- VPN/internal network is connected

### Docker fallback

Docker/Rancher Desktop can still be kept as a fallback, but it should be configured as a separate MCP server name such as:

```toml
[mcp_servers.jira2_docker]
enabled = false
```

Do not enable local and Docker versions for the same service at the same time unless testing. Codex will treat them as separate MCP servers and may show duplicate tools.

## References

- Codex MCP configuration: https://developers.openai.com/codex/mcp
- uv installation: https://docs.astral.sh/uv/getting-started/installation/
- MCP Atlassian: https://mcp-atlassian.soomiles.com/docs/installation
- GitHub MCP Server: https://github.com/github/github-mcp-server
