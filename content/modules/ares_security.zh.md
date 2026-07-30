---
title: "ares_security"
description: "无依赖的敏感数据脱敏器,含 9 种字段类型、JSON 感知脱敏与安全日志包装器。"
weight: 200
maturity: "Production"
---

# ares_security

## 职责

`ares_security` 在字符串与 JSON 载荷被记录或持久化之前,对其中的敏感信息进行脱敏。
它刻意保持无依赖:该包仅导入 `encoding/json`、`fmt`、`regexp` 与 `strings`,因此可
被任何工具或测试二进制 vendoring,而无需引入 ARES 的其余部分。

`Sanitizer` 内置针对 9 种敏感字段类型的正则模式,以及按类型保留少量上下文
(开头/结尾若干字符)的掩码函数,使日志仍有用但不泄露密钥。`SanitizeJSON` 会解析
JSON、对每个字符串值脱敏、再重新序列化,因此 JSON 结构不会被原始正则扫描破坏。
`SafeLogger` 包装任意 `func(string)` 日志器,使每条消息与格式参数都自动脱敏。

## 架构图

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

`NewSanitizer` 使用 `DefaultSanitizeOptions`;`NewSanitizerWithOptions` 接受自定义
`SanitizeOptions`(掩码字符、保留长度、按类型保留长度)。`SanitizeJSON` 解码到
`interface{}`,递归遍历 map/slice,对每个字符串应用 `Sanitize` 后再编码,若输入非
合法 JSON 则回退到普通 `Sanitize`。

## 外部接口

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

## 关键类型与方法

| 类型 / 方法 | 用途 |
|---|---|
| `SensitiveFieldType` | 9 种敏感字段类型的字符串枚举。 |
| `SensitivePattern` | 一种字段类型的正则 + 掩码函数 + 描述。 |
| `SanitizeOptions` | 掩码字符、保留长度标志、按类型保留长度。 |
| `Sanitizer` | 持有模式与选项;执行脱敏。 |
| `SafeLogger` | 包装 `func(string)`,对每条消息/参数脱敏。 |
| `NewSanitizer()` | 带默认模式与默认选项的脱敏器。 |
| `NewSanitizerWithOptions(opts)` | 带自定义选项的脱敏器。 |
| `Sanitize(input)` | 对普通字符串中的敏感数据脱敏。 |
| `SanitizeJSON(jsonStr)` | 解析、遍历、脱敏字符串、再序列化。 |
| `SanitizeLog(message)` | 包级一次性脱敏辅助。 |
| `NewSafeLogger(fn)` | 包装日志器以实现自动脱敏。 |
| `SafeLogger.Log` / `Logf` | 先脱敏再输出的消息辅助。 |

## 模块协作

- 无依赖:仅使用标准库的 `encoding/json`、`fmt`、`regexp`、`strings`。无内部
  ARES 导入。
- 被日志路径、仪表盘以及任何发出可能含密钥的面向用户/运维字符串的组件消费。
- `SafeLogger` 适配任意已有的 `func(string)` 日志器,因此可包装 `slog` 派生的
  辅助函数,而无需耦合特定日志器包。

## 扩展方式

1. 新增敏感类型:定义一个 `SensitiveFieldType` 常量,向传入 `NewSanitizer` 的列表
   追加一个 `SensitivePattern`(正则 + `MaskFunc`),并(可选)添加一个
   `PreserveLengthFor` 条目。
2. 自定义掩码行为:用 `DefaultSanitizeOptions()` 构建 `SanitizeOptions`,修改
   `MaskChar` 或 `PreserveLengthFor`,再传给 `NewSanitizerWithOptions`。
3. 用 `NewSafeLogger` 包装任意 `func(string)` 日志器,使所有 `Log`/`Logf` 输出
   自动脱敏。
4. 对于结构化载荷,调用 `SanitizeJSON` 而非 `Sanitize`,以在脱敏字符串值的同时
   保持 JSON 结构完整。
5. 使用包级 `SanitizeLog(message)` 进行一次性脱敏,无需实例化 `Sanitizer`。

## 双语状态

英文源为权威参考。中文页面保持相同的结构、签名与技术内容;两份页面中所有代码
标识符、类型名与字段类型常量均保持英文。

## 成熟度

`ares_security` 由 `sanitizer_test.go` 覆盖。它无依赖、已集成进日志/仪表盘路径,
API 稳定且无实验性标记。

{{< maturity "Production" >}}
