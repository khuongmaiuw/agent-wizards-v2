# Agent Wizards

<p align="center">
    <a href="https://github.com/khuongmaiuw/agent-wizards-v2"><img width="450" alt="Agent Wizards Logo" src="https://github.com/user-attachments/assets/cf8ca3ce-8b02-43f0-9d0f-5a331488da4b" /></a><br />
    <a href="https://github.com/khuongmaiuw/agent-wizards-v2/releases"><img src="https://img.shields.io/github/release/khuongmaiuw/agent-wizards-v2" alt="Latest Release"></a>
    <a href="https://github.com/khuongmaiuw/agent-wizards-v2/actions"><img src="https://github.com/khuongmaiuw/agent-wizards-v2/actions/workflows/build.yml/badge.svg" alt="Build Status"></a>
</p>

<p align="center">Your AI coding companion from UnicornWizard, now available in your favourite terminal.<br />Research, plan, and execute — wired into your LLM of choice.</p>

<p align="center"><img width="800" alt="Agent Wizards Demo" src="https://github.com/user-attachments/assets/58280caf-851b-470a-b6f7-d5c4ea8a1968" /></p>

## Features

- **Research → Plan → Code Pipeline:** use `--plan` flag or the `/plan` command in TUI to run a research agent, planner agent, and coder agent in sequence before touching your code
- **Multi-Model:** choose from a wide range of LLMs or add your own via OpenAI- or Anthropic-compatible APIs
- **Flexible:** switch LLMs mid-session while preserving context
- **Session-Based:** maintain multiple work sessions and contexts per project
- **LSP-Enhanced:** Agent Wizards uses LSPs for additional context, just like you do
- **Extensible:** add capabilities via MCPs (`http`, `stdio`, and `sse`)
- **Works Everywhere:** first-class support in every terminal on macOS, Linux, Windows (PowerShell and WSL), Android, FreeBSD, OpenBSD, and NetBSD

## Installation

Install with Go:

```bash
go install github.com/khuongmaiuw/agent-wizards-v2@latest
```

Or clone and build:

```bash
git clone https://github.com/khuongmaiuw/agent-wizards-v2.git
cd agent-wizards-v2
go build -o agent-wizards .
```

## Getting Started

The quickest way to get started is to grab an API key for your preferred
provider such as Anthropic, OpenAI, Groq, OpenRouter, or Vercel AI Gateway and just start
Agent Wizards. You'll be prompted to enter your API key.

That said, you can also set environment variables for preferred providers.

| Environment Variable        | Provider                                           |
| --------------------------- | -------------------------------------------------- |
| `ANTHROPIC_API_KEY`         | Anthropic                                          |
| `OPENAI_API_KEY`            | OpenAI                                             |
| `VERCEL_API_KEY`            | Vercel AI Gateway                                  |
| `GEMINI_API_KEY`            | Google Gemini                                      |
| `OPENROUTER_API_KEY`        | OpenRouter                                         |
| `GROQ_API_KEY`              | Groq                                               |
| `AWS_ACCESS_KEY_ID`         | Amazon Bedrock (Claude)                            |
| `AWS_SECRET_ACCESS_KEY`     | Amazon Bedrock (Claude)                            |
| `AWS_REGION`                | Amazon Bedrock (Claude)                            |
| `AZURE_OPENAI_API_ENDPOINT` | Azure OpenAI models                                |
| `AZURE_OPENAI_API_KEY`      | Azure OpenAI models                                |
| `AZURE_OPENAI_API_VERSION`  | Azure OpenAI models                                |

## Research → Plan → Code Pipeline

Agent Wizards includes a unique multi-agent pipeline that researches your codebase, creates a structured plan, then executes it.

### CLI

```bash
agent-wizards run --plan "add pagination to the users endpoint"
```

### TUI

1. Type your prompt in the input area
2. Press `alt+/` to open the Commands dialog (works even with text typed)
3. Select **"Run with Plan (Research → Plan → Code)"**

The pipeline runs three specialized agents in sequence:

- **Research agent** — read-only, explores your codebase and summarizes relevant context
- **Planner agent** — read-only, creates a numbered implementation plan saved to your todos
- **Coder agent** — executes the plan with full tool access

## Configuration

Agent Wizards runs great with no configuration. Configuration can be added either local to the project or globally, with the following priority:

1. `.agent-wizards.json`
2. `agent-wizards.json`
3. `$HOME/.config/agent-wizards/agent-wizards.json`

Configuration is stored as a JSON object:

```json
{
  "this-setting": { "this": "that" },
  "that-setting": ["ceci", "cela"]
}
```

Ephemeral data (application state) is stored in:

```bash
# Unix
$HOME/.local/share/agent-wizards/

# Windows
%LOCALAPPDATA%\agent-wizards\
```

### LSPs

```json
{
  "lsp": {
    "go": {
      "command": "gopls"
    },
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"]
    }
  }
}
```

### MCPs

```json
{
  "mcp": {
    "filesystem": {
      "type": "stdio",
      "command": "node",
      "args": ["/path/to/mcp-server.js"],
      "timeout": 120
    },
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/",
      "headers": {
        "Authorization": "Bearer $GH_PAT"
      }
    }
  }
}
```

### Ignoring Files

Agent Wizards respects `.gitignore` files by default. You can also create a `.crushignore` file to specify additional files and directories to ignore.

### Allowing Tools

```json
{
  "permissions": {
    "allowed_tools": [
      "view",
      "ls",
      "grep",
      "edit"
    ]
  }
}
```

### Disabling Built-In Tools

```json
{
  "options": {
    "disabled_tools": [
      "bash",
      "sourcegraph"
    ]
  }
}
```

### Agent Skills

Agent Wizards supports the [Agent Skills](https://agentskills.io) open standard. Skills are folders containing a `SKILL.md` file that Agent Wizards can discover and activate on demand.

Global skill paths:

* `$XDG_CONFIG_HOME/agents/skills` or `~/.config/agents/skills/`
* `$XDG_CONFIG_HOME/agent-wizards/skills` or `~/.config/agent-wizards/skills/`

Project-level skill paths:

* `.agents/skills`
* `.agent-wizards/skills`

```json
{
  "options": {
    "skills_paths": [
      "~/.config/agent-wizards/skills",
      "./project-skills"
    ]
  }
}
```

### Attribution Settings

```json
{
  "options": {
    "attribution": {
      "trailer_style": "co-authored-by",
      "generated_with": true
    }
  }
}
```

- `trailer_style`: `assisted-by` | `co-authored-by` | `none`
- `generated_with`: adds `✨ Generated with Agent Wizards` to commit messages

### Custom Providers

#### OpenAI-Compatible APIs

```json
{
  "providers": {
    "deepseek": {
      "type": "openai-compat",
      "base_url": "https://api.deepseek.com/v1",
      "api_key": "$DEEPSEEK_API_KEY",
      "models": [
        {
          "id": "deepseek-chat",
          "name": "Deepseek V3",
          "context_window": 64000,
          "default_max_tokens": 5000
        }
      ]
    }
  }
}
```

#### Anthropic-Compatible APIs

```json
{
  "providers": {
    "custom-anthropic": {
      "type": "anthropic",
      "base_url": "https://api.anthropic.com/v1",
      "api_key": "$ANTHROPIC_API_KEY",
      "models": [
        {
          "id": "claude-sonnet-4-20250514",
          "name": "Claude Sonnet 4",
          "context_window": 200000,
          "default_max_tokens": 50000,
          "can_reason": true,
          "supports_attachments": true
        }
      ]
    }
  }
}
```

### Local Models

#### Ollama

```json
{
  "providers": {
    "ollama": {
      "name": "Ollama",
      "base_url": "http://localhost:11434/v1/",
      "type": "openai-compat",
      "models": [
        {
          "name": "Qwen 3 30B",
          "id": "qwen3:30b",
          "context_window": 256000,
          "default_max_tokens": 20000
        }
      ]
    }
  }
}
```

## Logging

Logs are stored in `./.agent-wizards/logs/` relative to the project.

```bash
# Print the last 1000 lines
agent-wizards logs

# Follow logs in real time
agent-wizards logs --follow
```

Enable debug logging:

```json
{
  "options": {
    "debug": true
  }
}
```

## Keyboard Shortcuts

| Shortcut | Action |
|---|---|
| `ctrl+p` or `/` | Open commands dialog (when textarea empty) |
| `alt+/` | Open commands dialog (always, even with text typed) |
| `ctrl+n` | New session |
| `ctrl+s` | Sessions |
| `ctrl+l` | Switch model |
| `ctrl+t` | Toggle todos/task list (requires active session) |
| `ctrl+g` | Toggle help |
| `ctrl+c` | Quit |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) or open an issue on [GitHub](https://github.com/khuongmaiuw/agent-wizards-v2).

## License

[FSL-1.1-MIT](LICENSE.md)

---

Built by [UnicornWizard](https://github.com/khuongmaiuw) • Powered by the Charm ecosystem
