# Tool setup: Register the Microsoft Learn MCP server with Agent 365

Registers the public Microsoft Learn MCP server as a bring-your-own (BYO) MCP server in Microsoft Agent 365. After approval, calls go through the Agent 365 Tooling Gateway, so the server appears in the Microsoft 365 admin center, can be blocked or limited per tool, and requires a Microsoft Entra token even though the server itself has no authentication.

| Field | Value |
|---|---|
| Server URL | `https://learn.microsoft.com/api/mcp` |
| Server authentication | None (`NoAuth`) |
| Tools | microsoft_docs_search, microsoft_docs_fetch, microsoft_code_sample_search |
| Registration | Agent 365 CLI (`a365`), from a GitHub Codespace terminal (bash) |
| Approval | Microsoft 365 admin center > Agents > Tools > Requests (preview) |
| Status | Preview |

## Prerequisites

- A test tenant with Microsoft Agent 365 (for example Microsoft 365 E7).
- An account in the test tenant that can create app registrations. Approval and consent need the AI Administrator or Global Administrator role.
- The Agent 365 tools service principal (app ID `ea9ffc3e-8a23-4a7d-836d-234d7c7565c1`) in the tenant. A Global Administrator creates it once; see step 3.
- A GitHub Codespace. The commands below use bash.

## Choose a unique server name

Every registration needs its own name. Rules:

- Starts with `ext_`.
- Maximum 20 characters.
- Not used before in the tenant. A name from a failed attempt can stay blocked.

Example: `ext_Learn_AB` with your initials. Set it once per terminal session:

```bash
export SERVER_NAME="ext_Learn_AB"
export TENANT="<your-test-tenant>.onmicrosoft.com"
```

## 1. Check the tools in the Codespace

```bash
dotnet --list-sdks
az version
```

If `dotnet` is missing or older than 8.0:

```bash
curl -sSL https://dot.net/v1/dotnet-install.sh | bash /dev/stdin --channel 8.0
export PATH="$HOME/.dotnet:$PATH"
```

If `az` is missing:

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

## 2. Install the Agent 365 CLI

```bash
dotnet tool install --global Microsoft.Agents.A365.DevTools.Cli || dotnet tool update --global Microsoft.Agents.A365.DevTools.Cli
export PATH="$PATH:$HOME/.dotnet/tools"
a365 --version
```

## 3. Sign in to the test tenant

A Codespace has no local browser, so use the device code flow:

```bash
az login --tenant "$TENANT" --use-device-code --allow-no-subscriptions
az account show --query "{tenant:tenantId, user:user.name}" -o table
```

The Agent 365 CLI uses the tenant of this Azure CLI sign-in. There is no separate `a365 login`.

Check the Agent 365 tools service principal:

```bash
az ad sp show --id ea9ffc3e-8a23-4a7d-836d-234d7c7565c1 --query displayName -o tsv
```

The output is `Agent Tools`. If the command fails, ask a Global Administrator to run the one-time setup script described in [Set up service principal](https://learn.microsoft.com/microsoft-agent-365/developer/tooling#set-up-service-principal).

## 4. Read the tool names from the server

Tool names in the registration must exactly match the names the server exposes.

```bash
URL="https://learn.microsoft.com/api/mcp"
HDR=(-H "Content-Type: application/json" -H "Accept: application/json, text/event-stream")
SID=$(curl -s -D - -o /dev/null "${HDR[@]}" -X POST "$URL" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}' \
  | grep -i '^mcp-session-id:' | awk '{print $2}' | tr -d '\r')
curl -s "${HDR[@]}" ${SID:+-H "Mcp-Session-Id: $SID"} -X POST "$URL" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' \
  | sed -n 's/^data: //p' | jq -r '.result.tools[].name'
```

Expected output:

```text
microsoft_docs_search
microsoft_code_sample_search
microsoft_docs_fetch
```

If `jq` is missing, install it with `sudo apt-get install -y jq`.

## 5. Create the registration file

```bash
cat > learn-byo.json <<EOF
{
  "serverName": "$SERVER_NAME",
  "serverUrl": "https://learn.microsoft.com/api/mcp",
  "authType": "NoAuth",
  "description": "Microsoft Learn docs and code samples - demo BYO MCP",
  "publisherName": "Microsoft Learn",
  "tools": [
    { "name": "microsoft_docs_search",        "description": "Search official Microsoft and Azure documentation." },
    { "name": "microsoft_docs_fetch",         "description": "Fetch a Microsoft Learn page as markdown." },
    { "name": "microsoft_code_sample_search", "description": "Search code samples in Microsoft Learn." }
  ],
  "remoteScopes": null,
  "externalOAuth": null,
  "apiKey": null
}
EOF
cat learn-byo.json
```

The description has a maximum of 80 characters. Every tool needs a description.

## 6. Register the server

Run a dry run first, then the registration:

```bash
a365 develop-mcp register-external-mcp-server -f learn-byo.json --dry-run
a365 develop-mcp register-external-mcp-server -f learn-byo.json
```

1. Answer `y` to "Proceed with registration?".
2. Complete the sign-in when the CLI asks for it. In a Codespace, open the link that the terminal shows, or the forwarded port that VS Code offers, and sign in with the same test account.
3. Wait for the message `MCP server '<name>' has been registered successfully.`

If the sign-in in the Codespace doesn't complete, run steps 2 to 6 in a local terminal (Windows PowerShell or WSL) instead. The commands are the same.

## 7. Check the created objects

```bash
az ad app list --display-name "$SERVER_NAME" --query "[].{name:displayName, appId:appId}" -o table
```

| Object | Purpose |
|---|---|
| `<name>-A365Proxy` | Client app for the Copilot Studio connector. Users sign in through it. |
| `<name>-PublicClients` | Client app for VS Code, Claude Code, and GitHub Copilot CLI. |
| `<name> - BYO` (enterprise application) | The server as an Entra resource with the permission `Tools.ListInvoke.All`. |

## 8. Approve the server

1. Sign in to the Microsoft 365 admin center as AI Administrator or Global Administrator.
2. Go to Agents > Tools > Requests (preview).
3. Open the server, review the publisher and tools, select Approve, and grant consent.
4. After up to 30 minutes, the server appears under Agents > Tools > Registry and as a tool in Copilot Studio.

## 9. Use the server in Copilot Studio

1. Open an agent with the standard harness, for example the Cloud & AI Advisor.
2. Go to Tools > Add a tool > Model Context Protocol and search for the server name.
3. Create the connection and add the tool.
4. Test with "What admin role approves MCP server requests in the Microsoft 365 admin center?" and check the tool call in the activity map.

## 10. Govern the server

| Control | Where | Effect |
|---|---|---|
| Block the server | Admin center > Agents > Tools > Registry > server > Block | No agent in the tenant can call it |
| Turn off one tool | Registry > server > Tools tab | Only the selected tool is blocked, where the tab shows toggles |
| Limit users | Entra admin center > Enterprise applications > `<name>-A365Proxy` > Properties > Assignment required = Yes, then assign users | Only assigned users get a token in Copilot Studio |
| Limit environments | Power Platform admin center > connector policy | The connector is available only in allowed environments |
| Observe calls | Microsoft Defender > Advanced hunting | `CloudAppEvents \| where ActionType == "ExecuteToolByGateway"` |

## Clean up

A failed registration doesn't remove the Entra apps. Delete them before you try again with a new name:

```bash
az ad app list --display-name "$SERVER_NAME" --query "[].appId" -o tsv | xargs -r -n1 az ad app delete --id
```

To remove a registered server, delete it in the admin center under Agents > Tools > Registry, then run the command above.

## Known issues

| Message | Cause | Fix |
|---|---|---|
| `Tool description is required` | A tool in the file has no description | Add a description to every tool |
| `Short description exceeds the maximum length of 80 characters` | The server description is too long | Shorten `description` |
| `ApiDisplayNameIsInUse` | A connector from an earlier attempt uses the name | Use a new server name |
| Server not listed in Copilot Studio | Not approved yet, or the environment's connector policy blocks it | Approve, wait up to 30 minutes, check the environment |
