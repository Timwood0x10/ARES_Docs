---
title: "ares_config"
description: "YAML configuration loading, defaults, environment overrides, and validation for the ARES server."
weight: 170
maturity: "Production"
---

# ares_config

## Responsibility

`ares_config` owns the entire configuration surface for ARES. It reads a YAML
file, applies layered defaults, merges environment-variable overrides, and runs
a section-by-section validation pass before handing a fully populated `Config`
to the bootstrap layer. It also guards against path-traversal attacks when an
allowed config directory is configured.

The package defines every configuration struct the server understands: server,
LLM (with fallbacks and scorer rate limits), agents, tools, prompts, output,
validation, workflow, storage (with pgvector), memory (session, profile,
distillation, RAG, archive), knowledge (AKG retrieval), MCP servers, dashboard,
evolution (GA, LLM scoring, deployment), embedding, and discovery.

## Architecture

```mermaid
flowchart TD
    YAML[YAML file] --> Load
    Load[Load path] --> Sec[Path-traversal guard]
    Sec --> Read[os.ReadFile]
    Read --> Unmarshal[yaml.Unmarshal -> Config]
    Env[Environment variables] -.override.-> Unmarshal
    Unmarshal --> Defaults[setDefaults]
    Defaults --> Validate[Validate]
    Validate --> Sections[server/llm/agents/output/storage<br/>memory/knowledge/mcp/dashboard<br/>evolution/discovery]
    Sections --> Cfg[Validated *Config]
    Cfg --> Bootstrap[ares_bootstrap]
```

`Load` is the single entry point: it secures the path, parses YAML, calls
`setDefaults` to fill zero values with sensible defaults, then calls `Validate`.
`LoadFromEnv` is applied separately to let environment variables override the
YAML-derived values. Defaults are conditional: RAG, distillation, and knowledge
retrieval parameters are only filled when their respective enable flag is true,
so opting out keeps those subsystems inert.

## External interfaces

```go
// Load reads configuration from a YAML file, applies defaults, and validates it.
func Load(path string) (*Config, error)

// LoadFromEnv loads configuration from environment variables (overrides YAML).
func LoadFromEnv(cfg *Config) error

// SetAllowedConfigDir sets the allowed directory for config files (security).
func SetAllowedConfigDir(dir string)

// Validate validates the configuration values.
func (c *Config) Validate() error

// IsEnabled reports whether archiving is active (nil/true = enabled).
func (a ArchiveConfig) IsEnabled() bool

// DefaultArchiveDir is the default round-archive directory.
const DefaultArchiveDir = ".context/rounds"

// DefaultTaskDistillationPrompt is the default prompt for task distillation.
const DefaultTaskDistillationPrompt = "Please concisely summarize..."
```

## Key types and methods

| Type / Method | Purpose |
|---|---|
| `Config` | Top-level struct aggregating every config section. |
| `ServerConfig` | Host and port for the HTTP server. |
| `LLMConfig` | Provider, API key, model, timeouts, scorer rate/burst, fallbacks. |
| `AgentsConfig` / `LeaderConfig` / `SubAgentConfig` | Leader and sub-agent tuning. |
| `MemoryConfig` | Session, profile, distillation, RAG, archive settings. |
| `ArchiveConfig` | Round-archive dir and max rounds; `IsEnabled()` default-on semantics. |
| `KnowledgeConfig` | Optional AKG retrieval (TopK, MinScore). |
| `StorageConfig` / `PGVectorConfig` | Postgres/SQLite storage and pgvector. |
| `MCPConfig` / `MCPServerEntry` / `TransportEntry` | MCP server stdio/SSE transports. |
| `EvolutionConfig` | GA population, mutation, selection, deployment, LLM scoring. |
| `LLMScoringConfig` | Opt-in LLM-backed strategy scorer with seed and call budget. |
| `EmbeddingConfig` | Embedding client for experience distillation. |
| `DiscoveryConfig` | Optional service discovery engine. |
| `Load(path)` | Read, default, and validate a YAML config. |
| `LoadFromEnv(cfg)` | Overlay environment variables onto a config. |
| `setDefaults()` | Internal: conditional default filling. |
| `Validate()` | Internal: section-by-section validation. |

## Module collaboration

- Consumed by `ares_bootstrap` as the input to `Bootstrap`.
- `EvolutionConfig.Deployment` references `internal/evolution/deployment.DeploymentConfig`.
- Validation errors are wrapped via `internal/errors` before propagation.
- Memory and knowledge defaults feed `ares_memory` and the knowledge runtime so
  RAG/AKG retrieval only activates when explicitly enabled.

## Extension points

1. Add a new top-level section by defining a struct, adding a field to `Config`
   with a `yaml` tag, populating its defaults in `setDefaults`, and adding a
   `validate<Section>()` method called from `Validate`.
2. Add a new environment override by extending `LoadFromEnv` with a `os.Getenv`
   branch that maps to the target field.
3. Restrict config file locations by calling `SetAllowedConfigDir` at startup;
   `Load` then rejects any path that resolves outside that directory.
4. Opt into closed-loop memory by setting `memory.enable_rag: true`; the
   defaults layer fills `RAGTopK` (5) and `RAGMinScore` (0.4) only when enabled.
5. Opt into LLM strategy scoring by setting `evolution.llm_scoring.enabled: true`
   and tuning `max_calls_per_generation` to cap API cost per generation.

## Bilingual status

English source is canonical. The Chinese page mirrors structure, signatures,
and technical content; all code identifiers, type names, and YAML keys remain
in English in both pages.

## Maturity

`ares_config` is covered by `config_test.go`, `config_closed_loop_test.go`,
and `archive_config_test.go`. It is integrated into the SDK bootstrap path and
contains no experimental markers.

{{< maturity "Production" >}}
