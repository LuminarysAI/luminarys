# Luminarys

**Next-generation AI platform that combines an autonomous agent and an MCP server in a single runtime with sandboxed skill execution.**

Write skills in any language, run them in isolated WASM sandboxes with fine-grained permissions, expose via MCP or let the built-in agent orchestrate them autonomously.

[Documentation](https://luminarys.ai) | [Releases](https://github.com/LuminarysAI/luminarys/releases) | [Skill Examples](https://github.com/LuminarysAI/skill-examples) | [Demo Cluster](https://github.com/LuminarysAI/mcp-demo-cluster)

> The AI agent is currently in development. The MCP server is available for early evaluation.

## Why Luminarys

### WASM Skill Sandbox

Every skill runs in its own WebAssembly sandbox with fine-grained permissions. File system access, network, shell — everything is controlled by a deployment manifest. A skill cannot read a file, open a socket, or execute a command unless explicitly allowed. No containers, no VMs — just native-speed isolation at the function call level.

### Write Once, Run Anywhere

Skills compile to portable `.wasm` modules from AssemblyScript, Go, or Rust. The same binary runs on Linux, macOS, Windows — on x86, ARM, RISC-V, MIPS. No recompilation, no platform-specific builds.

### Multi-Node Clusters

Deploy skills across multiple nodes connected via NATS. The orchestrator routes calls transparently — a client sees one MCP server, but skills execute on the node that has them. File transfer between nodes is built in.

### MCP Protocol (Streamable HTTP + SSE + stdio)

Full MCP server with tool discovery, structured responses, and session management. Connect Claude Desktop, Cursor, Qwen CLI, MCP Inspector, or any MCP-compatible client. Skills appear as MCP tools automatically.

### Sub-Millisecond Dispatch

Skill invocation overhead is measured in microseconds. 

### Production Security

- DNS-aware network filtering (internal IPs allowed, external blocked without allowlist, cloud metadata always blocked)
- Glob patterns for filesystem permissions (`/data/projects/*/src:rw`)
- Shell command allowlists with directory restrictions
- HTTP and TCP allowlists with wildcard matching
- No skill can access another skill's data unless explicitly permitted via invoke policies

## How It Differs

| Feature | Luminarys | Typical MCP Server | Container-based Agent |
|---------|-----------|-------------------|----------------------|
| Skill isolation | WASM sandbox | Process / none | Docker container |
| Startup time | < 10ms | 100ms+ | Seconds |
| Multi-language | AS, Go, Rust (same ABI) | One language | Any (separate images) |
| Cross-platform binary | Single .wasm | Per-platform | Per-platform image |
| Clustering | Built-in (NATS) | Not available | Kubernetes required |
| Permission model | Per-skill manifest | Per-process | Per-container |

## Quick Start

Download the latest release for your platform from the [releases page](https://github.com/LuminarysAI/luminarys/releases). Extract and run:

```bash
./luminarys
```

The server starts on `http://localhost:8080` with a demo echo skill. Connect any MCP client to `http://localhost:8080/mcp`.

## Architecture

```
                         MCP Client (Claude Desktop, etc.)
                                    |
                              HTTP / SSE / stdio
                                    |
                         +----------+----------+
                         |     Luminarys Host   |
                         |                      |
                         |   Orchestrator       |
                         |   Session Manager    |
                         |   Permission Engine  |
                         +----+---------+-------+
                              |         |
                    +---------+--+  +---+--------+
                    | WASM Skill |  | WASM Skill  |  ...
                    | (sandbox)  |  | (sandbox)   |
                    +------------+  +-------------+
                              |
                         NATS (optional)
                              |
                    +---------+----------+
                    |   Remote Node      |
                    |   + WASM Skills    |
                    +--------------------+
```

## Demo

A multi-node cluster demo with 28+ skills across three languages is available at:

[github.com/LuminarysAI/mcp-demo-cluster](https://github.com/LuminarysAI/mcp-demo-cluster)

The demo runs via Docker Compose and showcases cross-node skill invocation, file transfer between nodes, and the full MCP protocol.

## Skill Development

Skills can be written in AssemblyScript, Go, or Rust. Example skills for all three languages are available in the [skill-examples](https://github.com/LuminarysAI/skill-examples) repository.

```bash
# Install the skill toolchain
lmsk genkey                          # generate signing key

# Build a skill (Rust example)
cd my-skill
lmsk generate -lang rust ./src      # generate dispatch code
cargo build --target wasm32-wasip1 --release
lmsk sign target/wasm32-wasip1/release/my_skill.wasm

# Inspect the result
lmsk info my.company.my-skill.skill
```

### SDKs

| Language | Package | Repository |
|----------|---------|------------|
| AssemblyScript | [@luminarys/sdk-as](https://www.npmjs.com/package/@luminarys/sdk-as) | [LuminarysAI/sdk-as](https://github.com/LuminarysAI/sdk-as) |
| Go | `github.com/LuminarysAI/sdk-go` | [LuminarysAI/sdk-go](https://github.com/LuminarysAI/sdk-go) |
| Rust | [luminarys-sdk](https://crates.io/crates/luminarys-sdk) | [LuminarysAI/sdk-rust](https://github.com/LuminarysAI/sdk-rust) |

## Configuration

```yaml
http:
  addr: ":8080"

skills:
  - config/skills/echo.yaml
  - config/skills/fs.yaml
```

Each skill has a deployment manifest:

```yaml
id: fs-skill
path: skills/ai.luminarys.rust.fs.skill
permissions:
  fs:
    enabled: true
    dirs: ["/data:rw"]
  shell: { enabled: false }
invoke_policy:
  can_invoke: []
  can_be_invoked_by: ["*"]
mcp:
  mapping: per_method
```

## License

The host runtime and `lmsk` toolchain are distributed under the [EULA](EULA.md).

SDKs are licensed under MIT.

Commercial licenses are available at [luminarys.ai](https://luminarys.ai).
