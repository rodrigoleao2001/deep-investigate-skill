# /deep-investigate -- Claude Code Skill for Datadog Support

A Claude Code custom slash command for deep ticket investigation with lab reproduction, structured evidence tracking, and adversarial cross-validation using Codex (ChatGPT CLI).

## What it does

When you run `/deep-investigate --ticket <URL|ID>` in Claude Code, it executes a multi-phase investigation:

1. **Intake**: parses the ticket, extracts key facts, classifies the issue
2. **Research**: parallel searches across Zendesk, Slack, Confluence/Jira KBs, source code, customer org telemetry, and public docs
3. **Claim Ledger**: maintains a structured hypothesis tracker with evidence states
4. **Lab Reproduction**: builds minimal reproductions in your personal Datadog org with Docker/Colima/cloud, tracks predictions vs results
5. **Codex Checkpoints**: sends sanitized (PII-free) summaries to Codex for adversarial review, runs multi-round debates to catch blind spots
6. **Output**: produces an internal memo (pt-BR by default) and a customer-facing reply (in the ticket's language), saved to disk and copied to clipboard

All outputs are **drafts only**. The skill never auto-posts to Zendesk, Slack, Jira, or any external system.

## Installation

### 1. Copy the skill files

```bash
# Clone this repo (or download the zip)
git clone <this-repo-url> /tmp/deep-investigate-skill

# Copy to your Claude Code commands directory
cp -r /tmp/deep-investigate-skill/commands/* ~/.claude/commands/
```

After this you should have:
```
~/.claude/commands/
  deep-investigate.md
  deep-investigate-refs/
    lab-experiment-contract.md
```

### 2. Configure your Datadog lab org

The skill needs a personal Datadog org where it can spin up lab reproductions. Open `~/.claude/commands/deep-investigate.md` and find the **"Configuracao do lab"** section. Replace the placeholders:

| Placeholder | What to put |
|---|---|
| `<YOUR_DATADOG_ORG_URL>` | Your personal Datadog org URL (e.g. `https://myname.datadoghq.com`) |
| `<YOUR_KEYCHAIN_API_KEY_SERVICE>` | The macOS Keychain service name where your DD API key is stored |
| `<YOUR_KEYCHAIN_APP_KEY_SERVICE>` | The macOS Keychain service name where your DD App key is stored |
| `<YOUR_KEYCHAIN_ACCOUNT>` | The macOS Keychain account name for both entries |

#### Storing keys in macOS Keychain

The skill retrieves credentials from macOS Keychain at runtime and injects them as environment variables. It never prints or logs them.

```bash
# Store your Datadog API key
security add-generic-password \
  -s "datadog-lab-api-key" \
  -a "your.name" \
  -w "YOUR_API_KEY_HERE" \
  -U

# Store your Datadog App key
security add-generic-password \
  -s "datadog-lab-app-key" \
  -a "your.name" \
  -w "YOUR_APP_KEY_HERE" \
  -U
```

Then in the skill file, set:
```
- API Key: `security find-generic-password -s "datadog-lab-api-key" -a "your.name" -w`
- App Key: `security find-generic-password -s "datadog-lab-app-key" -a "your.name" -w`
```

#### Linux alternative (no Keychain)

If you're on Linux, replace the `security find-generic-password` commands with whatever secret manager you use. For example, with `pass`:

```
- API Key: `pass show datadog/lab-api-key`
- App Key: `pass show datadog/lab-app-key`
```

Or with environment variables (less secure, keys visible in process list):

```
- API Key: read from $DD_LAB_API_KEY environment variable
- App Key: read from $DD_LAB_APP_KEY environment variable
```

### 3. Install required tools

The skill uses these tools during lab reproduction. Install what you need:

| Tool | Required for | Install |
|---|---|---|
| [Colima](https://github.com/abiosoft/colima) | Docker runtime on macOS | `brew install colima` |
| Docker CLI | Container-based reproductions | `brew install docker` |
| AWS CLI | AWS-based reproductions | `brew install awscli` |
| Azure CLI | Azure-based reproductions | `brew install azure-cli` |
| GCP CLI | GCP-based reproductions | `brew install --cask google-cloud-sdk` |
| Terraform | Infrastructure provisioning | `brew install terraform` |
| [Codex CLI](https://github.com/openai/codex) | Adversarial cross-validation | `npm install -g @openai/codex` |

Not all tools are needed for every investigation. The skill checks what's available during preflight and adapts.

### 4. Configure MCP servers (for full functionality)

The skill queries these MCP servers when available. Configure them in your Claude Code settings:

- **Zendesk MCP**: ticket lookup and search
- **Slack MCP**: internal channel search
- **Atlassian MCP**: Confluence/Jira KB access
- **Datadog MCP**: customer org telemetry (logs, traces, metrics, monitors)
- **SupportDog MCP**: org details, flare analysis

The skill works without all MCPs, it just skips the data sources that aren't available.

### 5. Optional: KB Tracker script

The skill can emit telemetry about which Confluence/Jira pages were accessed during research. If you want this, place a `kb-tracker.py` script at `~/.claude/kb-tracker.py`. If the script doesn't exist, the skill skips this step silently.

### 6. Optional: Customize language

By default, internal outputs (memo, claim ledger, notes) are in **pt-BR** and customer-facing replies match the ticket's language. To change the internal language, edit the **"Idioma"** section in the skill file.

## Usage

```bash
# Investigate a Zendesk ticket by URL
/deep-investigate --ticket https://datadog.zendesk.com/agent/tickets/1234567

# Investigate by ticket ID
/deep-investigate --ticket 1234567

# Focus on a specific area
/deep-investigate --ticket 1234567 --focus apm

# Resume a previous investigation
/deep-investigate --resume 1234567-20261021-1

# Paste ticket content directly (no MCP needed)
/deep-investigate
<paste ticket text>
```

## Output structure

Each investigation creates a directory at:
```
~/.claude/investigations/<ticket-id>-<YYYYMMDD>-<seq>/
  manifest.json           # Investigation metadata and state
  intake.json             # Parsed ticket data
  evidence/               # Individual evidence files (E1.md, E2.md, ...)
  claude-hypotheses-*.md  # Claude's initial hypotheses (before Codex)
  codex-review-*.md       # Codex review outputs
  codex-debate-*-round*.md    # Individual debate rounds
  codex-debate-*-consensus.md # Consensus summaries
  claim-ledger.md         # Hypothesis tracker
  lab/                    # Lab experiment artifacts
  memo-interno.md         # Internal investigation memo
  customer-reply.md       # Customer-facing reply draft
```

## How Codex cross-validation works

The skill uses [Codex CLI](https://github.com/openai/codex) (ChatGPT subscription, no API key needed) as an adversarial reviewer. At key checkpoints:

1. Claude sanitizes all data (replaces org names, IPs, hostnames with placeholders)
2. Sends the sanitized summary to Codex via `codex exec --sandbox read-only`
3. Codex produces an independent diagnosis or review
4. Claude and Codex debate point-by-point (2-5 rounds) until consensus or explicit disagreement
5. All rounds are saved to disk

This catches blind spots that a single model might miss. If Codex is unavailable (not installed, network issues), the skill logs it and continues without cross-validation.

## Adapting for your workflow

This skill was built for Datadog Technical Support Engineering. To adapt it for a different context:

1. **Product references**: replace Datadog-specific tools (SupportDog, Fleet Explorer, etc.) with your equivalents
2. **MCP servers**: swap for whatever data sources your team uses
3. **Lab setup**: point to your own test environment instead of a Datadog org
4. **Output format**: adjust the memo and reply templates to match your team's conventions
5. **Language**: change the "Idioma" section to your preferred language

## License

MIT
