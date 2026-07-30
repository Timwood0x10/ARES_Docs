---
title: "ares_security"
description: "Dependency-free sensitive-data sanitizer with 9 field types, JSON-aware redaction, and a safe logger wrapper."
weight: 200
maturity: "Production"
---

# ares_security

## Responsibility

`ares_security` redacts sensitive information from strings and JSON payloads
before they are logged or persisted. It is intentionally dependency-free: the
package imports only `encoding/json`, `fmt`, `regexp`, and `strings`, so it can
be vendored into any tool or test binary without pulling the rest of ARES.

The `Sanitizer` ships with regex patterns for nine sensitive field types and
per-type masking functions that preserve just enough context (first/last few
characters) to remain useful in logs without leaking the secret. `SanitizeJSON`
parses JSON, sanitizes every string value, and re-serializes, so JSON structure
is never corrupted by a raw regex pass. `SafeLogger` wraps any `func(string)`
logger so every message and format argument is sanitized automatically.

## Architecture

```mermaid
flowchart TD
    Input[String / JSON input] --> S[Sanitizer]
    S --> Patterns[defaultSensitivePatterns]
    Patterns --> P1[api_key]
    Patterns --> P2[password]
    Patterns --> P3[token]
    Patterns --> P4[email]
    Patterns --> P5[phone]
    Patterns --> P6[credit_card]
    Patterns --> P7[ssn]
    P1 --> Mask1[maskAPIKey]
    P2 --> Mask2[maskPassword]
    P3 --> Mask3[maskToken]
    P4 --> Mask4[maskEmail]
    P5 --> Mask5[maskPhone]
    P6 --> Mask6[maskCreditCard]
    P7 --> Mask7[maskSSN]
    S --> Sanitize[Sanitize string]
    S --> SanitizeJSON[SanitizeJSON: parse -> walk -> reserialize]
    Sanitize --> Out[Redacted string]
    SanitizeJSON --> Out
    SafeLogger[SafeLogger] --> S
    SafeLogger --> Underlying[func(string) logger]
```

`NewSanitizer` uses `DefaultSanitizeOptions`; `NewSanitizerWithOptions` accepts
a custom `SanitizeOptions` (mask char, keep-length, per-type preserve length).
`SanitizeJSON` decodes into `interface{}`, recursively walks maps/slices, and
applies `Sanitize` to every string before re-encoding, falling back to plain
`Sanitize` if the input is not valid JSON.

## External interfaces

```go
type SensitiveFieldType string

const (
    SensitiveFieldTypeAPIKey      SensitiveFieldType = "api_key"
    SensitiveFieldTypePassword    SensitiveFieldType = "password"
    SensitiveFieldTypeToken       SensitiveFieldType = "token"
    SensitiveFieldTypeSecret      SensitiveFieldType = "secret"
    SensitiveFieldTypeEmail       SensitiveFieldType = "email"
    SensitiveFieldTypePhone       SensitiveFieldType = "phone"
    SensitiveFieldTypeSSN         SensitiveFieldType = "ssn"
    SensitiveFieldTypeCreditCard  SensitiveFieldType = "credit_card"
    SensitiveFieldTypePersonalInfo SensitiveFieldType = "personal_info"
)

type SensitivePattern struct {
    Type        SensitiveFieldType
    Pattern     *regexp.Regexp
    MaskFunc    func(string) string
    Description string
}

type SanitizeOptions struct {
    KeepLength       bool
    MaskChar         rune
    PreserveLengthFor map[SensitiveFieldType]int
}

type Sanitizer struct{}

func NewSanitizer() *Sanitizer
func NewSanitizerWithOptions(options SanitizeOptions) *Sanitizer
func DefaultSanitizeOptions() SanitizeOptions

func (s *Sanitizer) Sanitize(input string) string
func (s *Sanitizer) SanitizeJSON(jsonStr string) string

func SanitizeLog(message string) string

type SafeLogger struct{}
func NewSafeLogger(underlying func(string)) *SafeLogger
func (l *SafeLogger) Log(message string)
func (l *SafeLogger) Logf(format string, args ...interface{})
```

## Key types and methods

| Type / Method | Purpose |
|---|---|
| `SensitiveFieldType` | String enum for the 9 sensitive field types. |
| `SensitivePattern` | Regex + mask function + description for one field type. |
| `SanitizeOptions` | Mask char, keep-length flag, per-type preserve length. |
| `Sanitizer` | Holds patterns + options; applies redaction. |
| `SafeLogger` | Wraps `func(string)` and sanitizes every message/arg. |
| `NewSanitizer()` | Default-pattern sanitizer with default options. |
| `NewSanitizerWithOptions(opts)` | Sanitizer with custom options. |
| `Sanitize(input)` | Redact sensitive data in a plain string. |
| `SanitizeJSON(jsonStr)` | Parse, walk, sanitize strings, re-serialize. |
| `SanitizeLog(message)` | Package-level one-shot sanitize helper. |
| `NewSafeLogger(fn)` | Wrap a logger for automatic sanitization. |
| `SafeLogger.Log` / `Logf` | Sanitize-then-emit message helpers. |

## Module collaboration

- Dependency-free: only `encoding/json`, `fmt`, `regexp`, `strings` from the
  standard library. No internal ARES imports.
- Consumed by logging paths, the dashboard, and any component that emits
  user-facing or operator-facing strings that may contain secrets.
- `SafeLogger` adapts to any existing `func(string)` logger, so it can wrap
  `slog`-derived helpers without coupling to a specific logger package.

## Extension points

1. Add a new sensitive type by defining a `SensitiveFieldType` constant,
   appending a `SensitivePattern` (regex + `MaskFunc`) to the list passed to
   `NewSanitizer`, and (optionally) a `PreserveLengthFor` entry.
2. Customize masking behavior by building `SanitizeOptions` with
   `DefaultSanitizeOptions()` and mutating `MaskChar` or `PreserveLengthFor`,
   then passing it to `NewSanitizerWithOptions`.
3. Wrap any `func(string)` logger with `NewSafeLogger` so all `Log`/`Logf`
   output is automatically redacted.
4. For structured payloads, call `SanitizeJSON` instead of `Sanitize` to keep
   JSON structure intact while redacting string values.
5. Use the package-level `SanitizeLog(message)` for one-shot redaction without
   instantiating a `Sanitizer`.

## Bilingual status

English source is canonical. The Chinese page mirrors structure, signatures,
and technical content; all code identifiers, type names, and field-type
constants remain in English in both pages.

## Maturity

`ares_security` is covered by `sanitizer_test.go`. It is dependency-free,
integrated into the logging/dashboard paths, and has a stable API with no
experimental markers.

{{< maturity "Production" >}}
