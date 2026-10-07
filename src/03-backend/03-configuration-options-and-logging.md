# Configuration, Options and Logging

Two unglamorous topics decide a lot about how an application behaves in production.
**Configuration** determines whether the same build can run in development, staging and
production, and whether secrets end up in Git. **Logging** determines whether, when
something goes wrong at 2 a.m., anyone can tell what happened.

Both look simple. Both have well-known failure modes: connection strings committed to
repositories, settings that silently fall back to defaults, log files filling the disk,
logs full of noise but missing the one fact you needed, and personal data written to
places it should never be.

---

## 1. The problem: one build, many environments

The same compiled application should run everywhere, with different:

- **connection strings** (local PostgreSQL vs a managed cloud database),
- **service URLs** (a fake mail server vs the real one),
- **feature flags and limits** (debug features on locally; strict rate limits in production),
- **secrets** (API keys, signing keys), which must never be in source control.

Rebuilding per environment is fragile ("it worked in staging" means nothing if staging ran
a different build). The principle, from the *Twelve-Factor App* methodology: **store
configuration in the environment, separate from code.**

---

## 2. The mental model: layered configuration providers

.NET configuration is a **merged view** over several **providers**, read in order. Later
providers override earlier ones, key by key:

```text
 1. appsettings.json                      ◄── defaults, committed
 2. appsettings.{Environment}.json        ◄── per-environment overrides, committed (no secrets)
 3. User secrets (Development only)       ◄── local secrets, outside the repo
 4. Environment variables                 ◄── set by the host, container or platform
 5. Command-line arguments                ◄── highest precedence
 (+ Azure Key Vault, App Configuration, etc. when added)
                         │
                         ▼
               IConfiguration: one key/value view
```

Keys are hierarchical, separated by `:`:

```json
// appsettings.json
{
  "ConnectionStrings": { "Beacon": "Host=localhost;Database=beacon;Username=beacon" },
  "Notifications": {
    "Channel": "log",
    "Smtp": { "Host": "localhost", "Port": 1025, "From": "beacon@localhost" }
  },
  "Sla": { "BreachCheckInterval": "00:01:00" }
}
```

`Notifications:Smtp:Port` is the key `1025`. In environment variables, where `:` isn't
allowed on all platforms, use a double underscore:

```bash
export Notifications__Smtp__Port=587
export ConnectionStrings__Beacon="Host=db.internal;Database=beacon;Username=beacon_app;Password=..."
```

### Environments

`ASPNETCORE_ENVIRONMENT` (or `DOTNET_ENVIRONMENT`) selects the environment: `Development`,
`Staging`, `Production` (the default if unset), or anything you name. It controls which
`appsettings.{Environment}.json` loads and is available as `app.Environment`.

> **⚠️ What can go wrong:** Because `Production` is the default, forgetting to set the
> environment on a developer machine gives you production behavior. Forgetting it on a
> server is safe; **setting `Development` on a server is dangerous**: it can enable the
> developer exception page, which leaks stack traces and internals.

---

## 3. The options pattern

Reading configuration with string keys everywhere (`config["Notifications:Smtp:Port"]`)
is fragile: typos fail silently, values are strings, and there's no validation. The
**options pattern** binds a configuration section to a typed class:

```csharp
public sealed class SmtpOptions
{
    public const string Section = "Notifications:Smtp";

    [Required] public string Host { get; init; } = "";
    [Range(1, 65535)] public int Port { get; init; } = 25;
    [Required, EmailAddress] public string From { get; init; } = "";
    public string? Username { get; init; }
    public string? Password { get; init; }
}
```

```csharp
builder.Services
    .AddOptions<SmtpOptions>()
    .BindConfiguration(SmtpOptions.Section)
    .ValidateDataAnnotations()
    .ValidateOnStart();            // fail at startup, not on the first email
```

```csharp
public sealed class EmailNotifier(IOptions<SmtpOptions> options) : INotifier
{
    private readonly SmtpOptions _smtp = options.Value;
    // ...
}
```

`ValidateOnStart` is the important line: a missing or invalid setting crashes the app
**at deployment**, where it's obvious, instead of hours later when the first email is sent.

### `IOptions`, `IOptionsSnapshot`, `IOptionsMonitor`

| Interface | Lifetime | Sees changes after startup? | Use in |
|---|---|---|---|
| `IOptions<T>` | Singleton | No | Most code |
| `IOptionsSnapshot<T>` | Scoped | Yes, per request | Scoped services wanting fresh values |
| `IOptionsMonitor<T>` | Singleton | Yes, with change notifications | Singletons that must react to reloads |

JSON files reload on change by default; environment variables don't change while a
process runs. Most apps only need `IOptions<T>`.

---

## 4. Secrets

Secrets (passwords, API keys, signing keys, connection strings with credentials) need
special handling.

- **Never in source control**, including `appsettings.Development.json`. Git history is
  forever (Book II, Chapter 1).
- **Locally**: **user secrets**, stored in your user profile, outside the repo:
  ```bash
  dotnet user-secrets init --project src/Beacon.Api
  dotnet user-secrets set "Notifications:Smtp:Password" "local-dev-password" --project src/Beacon.Api
  ```
  They're loaded automatically in Development. They're *not encrypted*; they just keep
  secrets out of Git.
- **In production**: a secrets manager such as **Azure Key Vault** (Book IX, Chapter 2),
  AWS Secrets Manager or HashiCorp Vault, ideally accessed with a **managed identity**, so
  there's no secret needed to fetch the secrets. Container platforms and Kubernetes also
  inject secrets as environment variables or mounted files.
- **Better still: no secret at all.** Managed identities let an Azure app authenticate to
  databases, storage and Key Vault without passwords.

> **🧱 Durable:** Treat every secret as something that *will* eventually leak, and design
> so that rotating it is easy and its blast radius is small: separate credentials per
> environment and per service, least privilege, and short-lived tokens where possible.

---

## 5. Logging: the mental model

`Microsoft.Extensions.Logging` separates **producing** log events from **storing** them:

```text
 Your code ──ILogger<T>──► Logging pipeline (filters by category + level) ──► Providers
                                                                              ├─ Console
                                                                              ├─ Debug
                                                                              ├─ OpenTelemetry → Application Insights, Grafana, ...
                                                                              └─ Serilog → files, Seq, Elasticsearch, ...
```

- **Category**: usually the class name (`ILogger<TicketService>` → `Beacon.Core.Tickets.TicketService`).
- **Level**: `Trace` < `Debug` < `Information` < `Warning` < `Error` < `Critical`.
- **Filtering** by category and level is configured in `appsettings.json`:

```json
"Logging": {
  "LogLevel": {
    "Default": "Information",
    "Microsoft.AspNetCore": "Warning",
    "Microsoft.EntityFrameworkCore.Database.Command": "Warning",
    "Beacon": "Debug"
  }
}
```

### Choosing levels

| Level | Use for | Example |
|---|---|---|
| Trace / Debug | Detailed diagnostics, usually off in production | "Cache miss for T-42" |
| Information | Significant normal events | "Ticket T-42 resolved by maria" |
| Warning | Unexpected but handled; might need attention | "Notification retry 2/3 for T-42" |
| Error | A failure for one operation | "Failed to send email for T-42" (with exception) |
| Critical | The application can't continue | "Cannot connect to database at startup" |

A useful test for `Error`: *should someone be woken up or create a ticket if this happens
a lot?* If not, it's probably `Warning` or `Information`.

---

## 6. Structured logging

The single most important logging practice. Compare:

```csharp
_logger.LogInformation($"Ticket {ticket.Id} assigned to {assignee}");          // ✗ string interpolation
_logger.LogInformation("Ticket {TicketId} assigned to {Assignee}", ticket.Id, assignee);  // ✓ message template
```

The second form keeps `TicketId` and `Assignee` as **named properties** alongside the
rendered message. Log stores (Application Insights, Seq, Elasticsearch, Loki) index them,
so you can query:

```text
TicketId = "T-42"                                   → the full history of one ticket
Assignee = "maria" and Level = "Error"              → every failure involving maria's tickets
```

With interpolation, you have only a string to grep. The interpolated version also
allocates the string even when the level is disabled.

Rules:

- **Use message templates** with PascalCase placeholders, never interpolation.
- **Keep property names consistent** (`TicketId` everywhere, not `Id` here and `ticket`
  there), so queries work across the codebase.
- **Pass the exception object** as the first argument for errors:
  `_logger.LogError(ex, "Failed to notify {Recipient}", recipient)`.
- For hot paths, use the **`[LoggerMessage]` source generator** (Book I, Chapter 12).

### Scopes and correlation

A **logging scope** attaches properties to every log entry within a block:

```csharp
using (_logger.BeginScope(new Dictionary<string, object> { ["TicketId"] = id }))
{
    // every log line here includes TicketId
}
```

ASP.NET Core automatically adds a **TraceId** (and SpanId) to every log entry during a
request. When a user reports an error, the `traceId` in the problem-details response
(Chapter 2) finds every log line for that request, and with distributed tracing, across
every service it touched (Book IX, Chapter 6).

---

## 7. What not to log

> **⚠️ What can go wrong:** Logs are copied to many places, retained for a long time, and
> read by many people. Never log passwords, tokens, API keys, full credit card numbers,
> or full request bodies that may contain them. Be careful with personal data (emails,
> names, addresses, IP addresses): regulations such as GDPR apply to logs too.

Practical defenses:

- Log **identifiers**, not content: `TicketId`, `UserId`, not the ticket body or user's email.
- Use redaction: `Microsoft.Extensions.Compliance.Redaction` can redact properties marked
  with data classification attributes; Beacon's `[Sensitive]` attribute (Book I, Chapter 12)
  is the same idea.
- Don't log entire objects with `{@Ticket}`-style destructuring unless you control what
  they contain.
- Review logging in code review like any other output.

---

## 8. In practice: configuring Beacon

### Typed options with validation

```csharp
// src/Beacon.Api/Options/BeaconOptions.cs
using System.ComponentModel.DataAnnotations;

namespace Beacon.Api.Options;

public sealed class NotificationOptions
{
    public const string Section = "Notifications";

    [Required, RegularExpression("^(log|email)$")]
    public string Channel { get; init; } = "log";
}

public sealed class SlaOptions
{
    public const string Section = "Sla";

    [Range(typeof(TimeSpan), "00:00:10", "01:00:00")]
    public TimeSpan BreachCheckInterval { get; init; } = TimeSpan.FromMinutes(1);
}
```

```csharp
// Program.cs (excerpt)
builder.Services.AddOptions<NotificationOptions>()
    .BindConfiguration(NotificationOptions.Section)
    .ValidateDataAnnotations()
    .ValidateOnStart();

builder.Services.AddOptions<SlaOptions>()
    .BindConfiguration(SlaOptions.Section)
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

Try setting `Sla__BreachCheckInterval=00:00:01` and starting the app: it fails immediately
with a clear validation message. That's the behavior you want in a deployment.

### Choosing the notifier from configuration

```csharp
builder.Services.AddSingleton<INotifier>(sp =>
    sp.GetRequiredService<IOptions<NotificationOptions>>().Value.Channel switch
    {
        "email" => ActivatorUtilities.CreateInstance<EmailNotifier>(sp),
        _       => ActivatorUtilities.CreateInstance<LoggingNotifier>(sp),
    });
```

### Structured, high-performance log messages

```csharp
// src/Beacon.Api/Log.cs
namespace Beacon.Api;

internal static partial class Log
{
    [LoggerMessage(EventId = 1001, Level = LogLevel.Information,
        Message = "Ticket {TicketId} created with priority {Priority}")]
    public static partial void TicketCreated(ILogger logger, string ticketId, string priority);

    [LoggerMessage(EventId = 1002, Level = LogLevel.Warning,
        Message = "Comment rejected for ticket {TicketId}: {ErrorCode}")]
    public static partial void CommentRejected(ILogger logger, string ticketId, string errorCode);
}
```

Used in endpoints (`ILogger<Program>` can be injected as a parameter):

```csharp
Log.TicketCreated(logger, ticket.Id.ToString(), ticket.Priority.ToString());
```

Note that we log the ID and priority, not the title. Titles are user content and may
contain personal data.

### JSON console logs for production

Locally, readable console output is best. In containers, logs are collected from stdout,
and JSON is far easier for log collectors to parse:

```json
// appsettings.Production.json
{
  "Logging": {
    "Console": { "FormatterName": "json", "FormatterOptions": { "IncludeScopes": true } }
  }
}
```

Book IX, Chapter 6 replaces this with OpenTelemetry export to Application Insights, which
adds traces and metrics alongside logs.

---

## 9. What can go wrong

- **Secrets in Git** via `appsettings*.json`.
- **Silent defaults**: a misspelled key means the default is used, and nobody notices.
  `ValidateOnStart` and required properties fix this.
- **`Development` in production**, exposing detailed errors.
- **String interpolation in logs**, losing structure.
- **Log noise**: `Information` for every cache hit, drowning the important events and
  costing money in log storage.
- **Missing context**: errors logged without the exception, IDs or trace ID.
- **Sensitive data in logs.**
- **Logging and rethrowing at every layer**, duplicating entries (Book I, Chapter 8).

---

## 10. How an experienced engineer thinks about this

- **Configuration is part of the deployable contract.** Validate it at startup; document
  every setting; keep defaults safe.
- **Secrets are a lifecycle**, not a value: where they're stored, who can read them, how
  they're rotated.
- **Logs are for answering questions.** Before adding a log line, ask what question it
  will help answer during an incident.
- **Structure over prose.** Queryable properties beat beautifully worded messages.
- **Logs are data with a cost and a risk**: storage costs money and log data can leak.

---

## 11. Check yourself

**Questions**

1. In what order are configuration providers applied by default? Which wins?
2. How do you set `Notifications:Smtp:Port` with an environment variable?
3. What does `ValidateOnStart` give you?
4. When would you use `IOptionsMonitor<T>` instead of `IOptions<T>`?
5. Where should secrets live locally, and in production?
6. Why are message templates better than string interpolation in logs?
7. What should never be logged?

**Exercises**

1. Add `SmtpOptions` with validation and an `EmailNotifier` that uses it (sending to a
   local test SMTP server such as MailHog or smtp4dev in Docker).
2. Run Beacon.Api with `ASPNETCORE_ENVIRONMENT=Production` locally and observe the
   differences (error responses, log format).
3. Add a logging scope with `TicketId` around all work in the comment endpoint and verify
   it appears on every related log line with `IncludeScopes`.
4. Search a project you know for interpolated log messages and rewrite three of them.

**Interview-style questions**

- "How do you manage configuration and secrets across environments?"
- "What's structured logging and why does it matter?"
- "How would you trace a single user's failed request through your logs?"

---

## 12. Going deeper

- [Microsoft docs: Configuration in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/configuration/)
- [Microsoft docs: Options pattern](https://learn.microsoft.com/dotnet/core/extensions/options)
- [Microsoft docs: Logging in .NET](https://learn.microsoft.com/dotnet/core/extensions/logging)
- [The Twelve-Factor App](https://12factor.net/) — especially "Config" and "Logs."

**Next:** [Chapter 4 — Designing REST APIs](04-designing-rest-apis.md) steps back from the
framework to the design of the API itself.
