---
type: write-up
title: Docker Agent
description: 코딩 없이 AI 에이전트 만들기
topics:
  - Docker
  - AI Agent
createdAt: 2026-11-02T18:57:00
---
Docker Agent is an open-source framework for building teams of specialized AI agents. Instead of prompting one generalist model, you define agents with specific roles and instructions that collaborate to solve problems. Run these agent teams from your terminal using any LLM provider.

> [!NOTE]
> Docker Agent is a framework for building and running custom agent teams. For Docker's built-in AI assistant, see Gordon

## Why agent teams

One agent handling complex work means constant context-switching. Split the work across focused agents instead - each handles what it's best at. Docker Agent manages the coordination.

Here's a two-agent team that debugs problems:
```yaml
agents:
	root:
		model: openai/gpt-6-luna
		description: Bug investigator
		instruction: |
			Analyze error messages, stack traces, and code to find bug root causes.
			Explain what's wrong and why it's happening
			Delegate fix implementation to the fixer agent.
		sub_agents: [fixer]
		toolsets:
			- type: filesystem
			- type: mcp
			  ref: "docker:duckduckgo"
	fixer:
		model: anthropic/claude-sonnet-5
		description: Fix implementer
		instruction: |
			Write fixes for bugs diagnosed by the investigator.
			Make minimal, targeted changes and add tests to prevent regression.
		toolsets:
			- type: filesystem
			- type: shell
```
The root agent investigates and explains the problem. When it understands the issue, it hands off to `fixer` for implementation. Each agent stays focused on its specialty.

## Installation

Docker Agent is included in Docker Desktop 4.63 and later. In Docker Desktop versions 4.49 through 4.62, this feature was called cagent.

For Docker Engine users or custom installations:
- **Homebrew**: `brew install docket-agent`
- **Winget**: `winget install Docker.Agent`
- **Pre-built binaries**: GitHub releases
- **From source**: See the Docker Agent repository

The `docker-agent` binary should be copied to `~/.docker/cli-plugins` and then can be used with the `docker agent` command. Alternatvely, it can be used as a standalone binary.

## Get started
### Set your API key for the model provider you want to use:

```sh
export ANTHROPIC_API_KEY=<key>
export OPENAI_API_KEY=<key>
export GOOGLE_API_KEY=<key>
```

### Save the example configuratino as `debugger.yaml`

```yaml
agents:
	root:
		model: openai/gpt-6-luna
		description: Bug investigator
		instruction: |
			Analyze error messages, stack traces, and code to find bug root causes.
			Explain what's wrong and why it's happening
			Delegate fix implementation to the fixer agent.
		sub_agents: [fixer]
		toolsets:
			- type: filesystem
			- type: mcp
			  ref: "docker:duckduckgo"
	fixer:
		model: anthropic/claude-sonnet-5
		description: Fix implementer
		instruction: |
			Write fixes for bugs diagnosed by the investigator.
			Make minimal, targeted changes and add tests to prevent regression.
		toolsets:
			- type: filesystem
			- type: shell
```

### Run your agent team

```sh
docker agent run debugger.yaml
```

You'll see a prompt where you can describe bugs or paste error messages. The investigator analyzes the problem, then hands off to the fixer for implementation.

## How it works

You interact with the root agent, which can delegate work to sub-agents you define. Each agent:
- Uses its own model and parameters
- Has its own context
- Can access built-in tools like todo lists, memory, and task delegation
- Can use external tools via MCP servers
The root agent delegates tasks to agents listed under `sub_agents`. Sub-agents can have their own sub-agents for deeper hierachies.

## Configuration options

Agent configurations are YAML files. A basic structure looks like this:
```yaml
agents:
	root:
		model: claude-sonnet-5
		description: Brief role summary
		instruction: |
			Detailed instructions for this agent...
		sub_agents: [ helper ]
	helper:
		model: gpt-6-luna
		description: Specialist agent role
		instruction: |
			Instructions for the helper agent...
```

You can also configure model settings (like context limits), tools (including MCP servers), and more. See the configuration reference for complete details.

## Share agent teams

Agent configurations are packaged as OCI artifacts. Push and pull them like container images:

```sh
docker agent share push ./debugger.yaml username/debugger
docker agent share pull username/debugger
```

Use Docker Hub or any OCI-compatible registry. Pushing creates the repository if it doesn't exist yet.
