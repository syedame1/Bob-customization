# dockerfile-generator

A Bob subagent (dockerfile-generator) that analyzes your project and generates production-ready Dockerfiles automatically.

---

## What it does

- Detects your tech stack by reading project files (`package.json`, `requirements.txt`, `go.mod`, `pom.xml`, `Cargo.toml`, etc.)
- Generates a Dockerfile following production best practices (multi-stage builds, non-root user, layer caching, slim base images)
- Writes a recommended `.dockerignore` alongside the Dockerfile
- Explains every key decision it made

---

## Setup

### 1. Create the agents directory

```bash
# Project-level (shared with your team via git)
mkdir -p .bob/agents

# OR user-level (available across all your projects)
mkdir -p ~/.bob/agents
```

### 2. Create the subagent file

Create `.bob/agents/dockerfile-generator.md` (or `~/.bob/agents/dockerfile-generator.md`) with the following content:

```markdown
---
name: dockerfile-generator
description: Use this agent when you need to generate, review, or optimize a Dockerfile for any language or framework. Invoke proactively when the user mentions Docker, containerization, or building an image.
tools: Read, Write, Glob, Bash
model: sonnet
---

You are a Docker specialist focused on generating production-ready Dockerfiles.

## Your job

When given a project directory or tech stack description, produce a Dockerfile that:

1. **Chooses the right base image** — use official slim/alpine variants where possible (e.g. `node:20-alpine`, `python:3.12-slim`)
2. **Uses multi-stage builds** for compiled languages or when build tooling should not ship in the final image
3. **Minimizes layers** — combine RUN commands with `&&`, clean up caches in the same layer
4. **Runs as a non-root user** — always add a dedicated user and `USER` instruction
5. **Sets a WORKDIR** early
6. **Copies dependency files before source** so layer caching is maximized
7. **Exposes the correct port** and sets a sensible `CMD` or `ENTRYPOINT`
8. **Includes a .dockerignore recommendation** alongside the Dockerfile

## Workflow

1. Read key project files to detect the stack automatically
2. Ask clarifying questions only if the runtime, entry point, or port are genuinely ambiguous
3. Write the Dockerfile to the project root (or path the user specifies)
4. Print a brief explanation of the key decisions made

## Output format

- The Dockerfile itself (written to disk)
- A short summary: base image chosen, why multi-stage (or not), port, user, and any caveats
- A recommended `.dockerignore` content block
```


## Usage

### Automatic (recommended)

Just describe what you want — Bob will delegate to the subagent automatically:

```
Containerize this app
```

```
Add Docker support to this project
```

### Explicit @-mention

```
@agent-dockerfile-generator Generate a Dockerfile for this project
```

### With extra context (best results)

```
@agent-dockerfile-generator Analyze this project and generate a production-ready
Dockerfile. The app runs on port 3000 and the entry point is src/index.js.
Write the Dockerfile to the project root and include a .dockerignore file.
```


---

## Configuration

The subagent's frontmatter controls its behavior. Edit the `.md` file to tune it:

| Field | Default | Notes |
|---|---|---|
| `description` | *(see above)* | Controls when Claude auto-invokes it — be specific |
| `tools` | `Read, Write, Glob, Bash` | Add `WebFetch` to let it look up current base image tags |

---

## File structure

```
your-project/
└── .bob/
    └── agents/
        └── dockerfile-generator.md   ← the subagent definition
```

Or globally:

```
~/.bob/
└── agents/
    └── dockerfile-generator.md
```

---

## Tips

- **You don't need to describe your stack** — the agent reads your project files automatically.
- Subagents run in their own context window — your main session stays clean and focused.
