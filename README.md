# AI-Assisted SOC Monitoring Lab with Splunk Enterprise, MCP Server, and Claude Desktop

## Project Overview

This lab extended an existing Splunk Enterprise environment by adding Windows security monitoring dashboards and integrating the official Splunk MCP Server with Claude Desktop.

The goal was to create an end-to-end workflow where Windows Event Logs could be collected, visualized, searched with SPL, and investigated through natural-language requests sent from Claude Desktop to Splunk through the Model Context Protocol (MCP).

The project also required troubleshooting several Windows-specific issues involving Node.js, `npx`, Claude Desktop, command execution, TLS certificates, and authorization headers.

---

## Project Architecture

```text
WindowsAgent Virtual Machine
        |
        | Splunk Universal Forwarder
        | TCP 9997
        v
Splunk Enterprise
        |
        | Windows Event Logs
        | Dashboards and SPL searches
        v
Splunk MCP Server
        |
        | HTTPS / Management API
        | Port 8089
        v
Claude Desktop
```

---

## Technologies Used

- Splunk Enterprise
- Splunk Universal Forwarder
- Splunk MCP Server
- Claude Desktop
- Windows Event Viewer
- Windows PowerShell
- Node.js and npm
- `npx`
- `mcp-remote`
- SPL
- JSON configuration
- Windows Defender Firewall
- VirtualBox Windows virtual machine

---

## Lab Objectives

1. Confirm that Windows Event Logs from the `WindowsAgent` virtual machine were being received by Splunk Enterprise.
2. Create a security-monitoring dashboard using live Windows data.
3. Install and configure the official Splunk MCP Server.
4. Create an encrypted MCP authentication token.
5. Grant the required MCP execution and administration capabilities.
6. Connect Claude Desktop to the Splunk MCP endpoint.
7. Use natural-language prompts to search Splunk data.
8. Troubleshoot Windows command execution, TLS, and authorization-header problems.
9. Validate that Claude could execute Splunk searches and return real log results.

---

## Windows Log Sources

The Splunk Universal Forwarder collected the following Windows Event Logs:

```ini
[WinEventLog://Application]
disabled = 0
index = main
renderXml = false

[WinEventLog://System]
disabled = 0
index = main
renderXml = false

[WinEventLog://Security]
disabled = 0
index = main
renderXml = false
```

The Windows host appeared in Splunk as:

```text
WindowsAgent
```

A verification search was used to confirm the available sources and sourcetypes:

```spl
index=* earliest=0 host=WindowsAgent
| stats count by source sourcetype
| sort - count
```

The received data included:

- `WinEventLog:Security`
- `WinEventLog:System`
- `WinEventLog:Application`
- A test log from `C:\splunk-test\test.log`

---

## Security Dashboard

A dashboard named **WindowsAgent Security Monitoring** was created in Splunk.

### Events Over Time

```spl
index=* host=WindowsAgent
| timechart span=30m count
```

Purpose:

- Visualize event volume over time
- Identify sudden increases or gaps in log activity
- Confirm that events are being received

### Top Security EventCodes

```spl
index=* host=WindowsAgent source="WinEventLog:Security"
| stats count by EventCode
| sort - count
| head 10
```

Purpose:

- Identify the most common Windows Security Event IDs
- Highlight authentication, privilege, and account-related activity

### Failed Logons

```spl
index=* host=WindowsAgent source="WinEventLog:Security" EventCode=4625
| stats count by Account_Name Logon_Type Failure_Reason
| sort - count
```

Purpose:

- Identify repeated failed authentication attempts
- Review affected accounts and failure reasons
- Support brute-force and password-spraying investigations

### Successful Logons

```spl
index=* host=WindowsAgent source="WinEventLog:Security" EventCode=4624
| stats count by Account_Name Logon_Type
| sort - count
```

### Privileged Logons

```spl
index=* host=WindowsAgent source="WinEventLog:Security" EventCode=4672
| stats count by Account_Name
| sort - count
```

### Account Changes

```spl
index=* host=WindowsAgent source="WinEventLog:Security"
(EventCode=4720 OR EventCode=4726 OR EventCode=4732)
| table _time EventCode Subject_Account_Name Target_Account_Name Message
| sort - _time
```

### Recent Raw Activity

```spl
index=* host=WindowsAgent
| table _time source sourcetype EventCode Account_Name Message
| sort - _time
| head 20
```

---

## Dashboard Time-Range Troubleshooting

Some dashboard panels initially returned no results because their searches used hardcoded time restrictions such as:

```spl
earliest=-24h
```

The Windows events in the lab were older than the selected time range.

The solution was to:

- Remove hardcoded `earliest` values from panel searches
- Set each panel data source to **Default**
- Allow the dashboard global time picker to control the searches
- Use **All time** while troubleshooting historical data

One panel also had a static 30-minute time range, which caused it to appear blank. Changing the panel time configuration from **Static** to **Default** corrected the issue.

---

## Splunk MCP Server Installation

The official Splunk MCP Server application was installed directly on the Splunk Enterprise system.

```text
Splunk Web
→ Apps
→ Manage Apps
→ Install app from file
→ Upload the Splunk MCP Server package
```

The MCP Server was installed on the Splunk Enterprise laptop, not on the Windows virtual machine.

---

## MCP Permissions

The required MCP capabilities were enabled:

```text
mcp_tool_admin
mcp_tool_execute
```

---

## MCP Token and Endpoint

An encrypted MCP token was created inside the Splunk MCP Server application.

The local endpoint used was:

```text
https://localhost:8089/services/mcp
```

Security requirements:

- Never include the real token in screenshots
- Never commit the token to GitHub
- Replace the token with a placeholder in documentation
- Revoke and regenerate any token exposed during troubleshooting

---

## Claude Desktop Integration

Claude Desktop was configured to launch `mcp-remote`, which acted as a local bridge between Claude and the Splunk MCP Server.

The final working configuration followed this structure:

```json
{
  "mcpServers": {
    "splunk": {
      "command": "C:\\Windows\\System32\\cmd.exe",
      "args": [
        "/c",
        "npx",
        "-y",
        "mcp-remote",
        "https://localhost:8089/services/mcp",
        "--header",
        "Authorization:${AUTH_HEADER}"
      ],
      "env": {
        "AUTH_HEADER": "Bearer YOUR_SPLUNK_MCP_TOKEN",
        "NODE_TLS_REJECT_UNAUTHORIZED": "0"
      }
    }
  }
}
```

`YOUR_SPLUNK_MCP_TOKEN` is a placeholder and must be replaced locally with the encrypted token generated by Splunk.

---

## Troubleshooting Summary

### Issue 1: `npx` Was Not Recognized

Initial error:

```text
'npx' is not recognized as an internal or external command
spawn npx ENOENT
```

Cause:

- Node.js and npm were not available in Claude Desktop's process environment
- Claude could not locate `npx`

Resolution:

```powershell
node --version
npx --version
```

After Node.js was installed and Claude Desktop was restarted, its process environment included the Node.js and npm paths.

---

### Issue 2: The Spawn Bug

This was the main technical issue.

Claude Desktop on Windows resolved `npx` to its full path:

```text
C:\Program Files\nodejs\npx.cmd
```

Claude then passed that path to `cmd.exe` without correctly wrapping it in quotation marks.

Because the path contains a space in `Program Files`, `cmd.exe` split the command at the space and attempted to execute:

```text
C:\Program
```

This produced:

```text
'C:\Program' is not recognized as an internal or external command
```

The problem occurred even when the command field contained `npx` or the complete path because Claude Desktop performed its own internal command resolution and passed the resolved path incorrectly.

#### Final Fix

The fix was to explicitly invoke `cmd.exe` and pass `npx` as a separate, bare argument:

```json
"command": "C:\\Windows\\System32\\cmd.exe",
"args": [
  "/c",
  "npx",
  "-y",
  "mcp-remote"
]
```

This bypassed Claude Desktop's broken internal path-quoting behavior. `cmd.exe` received `npx` as a normal command and resolved it through the Windows PATH.

---

### Issue 3: TLS Certificate Warning

The Splunk management endpoint used HTTPS with a local self-signed certificate.

Node.js displayed:

```text
Setting the NODE_TLS_REJECT_UNAUTHORIZED environment variable to '0'
makes TLS connections and HTTPS requests insecure
```

For this isolated local lab, certificate verification was disabled with:

```json
"NODE_TLS_REJECT_UNAUTHORIZED": "0"
```

This setting should not be used in production.

---

### Issue 4: Authorization Header Bug

After the spawn problem was fixed, `mcp-remote` started successfully but reported:

```text
Warning: ignoring invalid header argument: "Authorization:
```

The header had accidentally been merged or malformed as one large string, causing the authorization value to be truncated or parsed incorrectly.

This caused:

- The Splunk token not to be sent correctly
- `mcp-remote` to attempt OAuth discovery
- The server to return HTTP 405 errors

Example:

```text
HTTP 405 Method Not Allowed
```

#### Final Fix

The header was restored as a clean, separate array item:

```json
"--header",
"Authorization:${AUTH_HEADER}"
```

The token value was stored separately:

```json
"AUTH_HEADER": "Bearer YOUR_SPLUNK_MCP_TOKEN"
```

This prevented Windows spacing and parsing problems.

---

## Understanding the HTTP 405 Error

The HTTP 405 response did not mean that the Splunk MCP endpoint was missing.

It appeared because the authorization header was malformed or ignored. Without the valid bearer token, `mcp-remote` attempted a different authentication flow and used unsupported request methods against the endpoint.

Once the header was correctly passed, the MCP client could authenticate normally.

---

## Validation

The integration was considered successful after Claude Desktop could:

- Connect to the Splunk MCP Server
- Access the tools exposed by Splunk
- Execute searches against Splunk Enterprise
- Retrieve real events from `WindowsAgent`
- Return SPL queries and summarize search results

Example validation prompt:

```text
Use the Splunk MCP server to count events from host WindowsAgent
by source and sourcetype. Show the SPL query and summarize the results.
```

Expected SPL:

```spl
index=* host=WindowsAgent
| stats count by source sourcetype
| sort - count
```

A second validation prompt:

```text
Use the Splunk MCP server to identify the 10 most common
Windows Security EventCodes from WindowsAgent.
Show the SPL and explain what each EventCode represents.
```

Expected SPL:

```spl
index=* host=WindowsAgent source="WinEventLog:Security"
| stats count by EventCode
| sort - count
| head 10
```

Claude successfully assisted with searching logs and retrieving entries from the Splunk server.

---

## Example Investigation Prompts

### Failed Logons

```text
Use the Splunk MCP server to find failed Windows logons from
WindowsAgent. Show the account, logon type, failure reason,
source IP when available, event count, and SPL query used.
```

### Privileged Activity

```text
Use Splunk to find EventCode 4672 activity on WindowsAgent.
Show the accounts involved and summarize the results.
```

### Account Creation

```text
Use Splunk to find newly created accounts on WindowsAgent
using EventCode 4720. Show the target account, subject account,
timestamp, and SPL query.
```

### General SOC Review

```text
Act as a SOC analyst and review WindowsAgent for suspicious activity.
Show all SPL searches used, explain the evidence, and do not classify
activity as malicious without supporting events.
```

---

## Key Lessons Learned

- Installing an MCP server is only one part of the integration; the client, authentication, command runtime, and transport must also work.
- Windows path handling can break commands when executable paths contain spaces.
- Explicitly separating the shell, executable, and arguments can solve command-spawn problems.
- Authorization headers should be passed as separate arguments instead of being merged into one command string.
- A valid endpoint can still return HTTP 405 when the client uses the wrong authentication flow.
- Self-signed certificates require special handling in a local lab.
- Dashboard time ranges can make valid data appear missing.
- AI-generated searches should be reviewed and validated by the analyst.
- Secrets must be removed before publishing configurations or screenshots.

---

## Security Considerations

Before sharing the project publicly:

- Revoke any token exposed during troubleshooting
- Generate a new MCP token
- Replace all tokens with placeholders
- Do not expose Splunk port 8089 to the public internet
- Do not disable TLS verification in production
- Review AI-generated SPL before relying on its conclusions

---

## Project Outcome

The completed lab demonstrated an end-to-end AI-assisted SOC workflow:

```text
Windows Event Logs
        ↓
Splunk Universal Forwarder
        ↓
Splunk Enterprise
        ↓
Security Dashboards and SPL
        ↓
Splunk MCP Server
        ↓
Claude Desktop
        ↓
Natural-Language Log Investigation
```

The final result was a functioning Splunk environment where Claude Desktop could assist with log discovery, SPL execution, Windows security-event analysis, and SOC investigation tasks.

---

## Skills Demonstrated

- Splunk Enterprise administration
- Splunk Universal Forwarder configuration
- Windows Event Log ingestion
- SPL search development
- SIEM dashboard creation
- MCP server installation
- Claude Desktop MCP integration
- JSON configuration
- Node.js and npm troubleshooting
- Windows process and PATH troubleshooting
- HTTPS and certificate troubleshooting
- Bearer-token authentication
- SOC investigation workflow
- Technical documentation
- Root-cause analysis

---

## Resume Project Description

**AI-Assisted SOC Monitoring Lab with Splunk Enterprise and Claude MCP**

Built a Windows security-monitoring environment using Splunk Enterprise and Universal Forwarder, created dashboards for authentication and account activity, and integrated the Splunk MCP Server with Claude Desktop for natural-language log investigation. Resolved Windows command-spawn, path-quoting, TLS, and authorization-header issues while validating AI-generated SPL against live Windows Event Logs.
