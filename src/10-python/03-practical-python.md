# Practical Python

Chapters 1 and 2 covered the language and the tooling. This chapter is about what you'll
actually use Python for as a .NET engineer: calling APIs, validating and transforming data,
writing command-line tools and automation scripts, and preparing data for AI work. Each
section builds a piece of `beacon-tools`, which Book XI then uses for AI evaluation and
document processing.

---

## 1. The problem: glue work

Real systems are surrounded by small jobs that don't belong in the main application:

- export tickets to CSV for a manager's spreadsheet,
- bulk-import knowledge-base articles from another system,
- run an evaluation of AI answers against a test set (Book XI, Chapter 8),
- check that every customer has a valid SLA configuration,
- generate test data for load tests,
- clean up orphaned attachments.

Python excels at this kind of work: concise, a vast library ecosystem, and fast to iterate.
The difference between a fragile script and a reliable tool is applying the same engineering
discipline as in Beacon's main code: typed inputs, validation, error handling, logging, tests.

---

## 2. The mental model: scripts are software too

```text
 Untrusted input ─► validate (Pydantic) ─► typed objects ─► transform ─► output
 (API JSON, CSV,                                                       (API calls, files,
  user arguments)                                                       reports)
        ▲                                                                   │
        └──────── errors reported clearly, exit codes, logs ◄───────────────┘
```

The same "parse, don't validate" boundary from Book III, Chapter 5 and Book V, Chapter 4
applies: convert untrusted data into validated types at the edge, then work with types you
trust.

---

## 3. Calling APIs with httpx

**httpx** is the modern HTTP client: sync and async APIs, HTTP/2, timeouts, connection pooling.
(The older `requests` library is still everywhere and works fine for simple sync scripts.)

```python
# src/beacon_tools/client.py
from __future__ import annotations

import os
from collections.abc import AsyncIterator

import httpx

from .schemas import Page, TicketSummary


class BeaconClient:
    """Async client for Beacon's API, authenticated with a bearer token (service-to-service)."""

    def __init__(self, base_url: str, token: str, *, timeout: float = 15.0) -> None:
        self._http = httpx.AsyncClient(
            base_url=base_url,
            headers={"Authorization": f"Bearer {token}", "Accept": "application/json"},
            timeout=httpx.Timeout(timeout, connect=5.0),
            transport=httpx.AsyncHTTPTransport(retries=2),      # connection-level retries only
        )

    async def __aenter__(self) -> BeaconClient:
        return self

    async def __aexit__(self, *exc: object) -> None:
        await self._http.aclose()

    async def list_tickets(self, **filters: str | int) -> AsyncIterator[TicketSummary]:
        """Yield every ticket, following cursor pagination (Book III, Chapter 4)."""
        cursor: str | None = None
        while True:
            params = {**filters, "limit": 100, **({"cursor": cursor} if cursor else {})}
            response = await self._http.get("/api/tickets", params=params)
            response.raise_for_status()
            page = Page[TicketSummary].model_validate(response.json())
            for item in page.items:
                yield item
            if not page.next_cursor:
                return
            cursor = page.next_cursor

    @classmethod
    def from_env(cls) -> BeaconClient:
        return cls(os.environ["BEACON_API_URL"], os.environ["BEACON_API_TOKEN"])
```

Points:

- **Always set timeouts** (Book III, Chapter 1); httpx defaults to 5 seconds, `requests` to
  none at all.
- **One client per run**, reused for connection pooling (the `HttpClient` lesson again).
- **An async generator** (`async def` + `yield`) hides pagination: callers just
  `async for ticket in client.list_tickets(status="Open")`, like C#'s `IAsyncEnumerable<T>`
  (Book I, Chapter 9).
- **Credentials from the environment**, never hard-coded. For Azure resources, the
  `azure-identity` package provides `DefaultAzureCredential` in Python too (Book IX,
  Chapter 2).

For retries on HTTP errors (429, 503) with back-off, use **tenacity** or `httpx` transport
hooks, honoring `Retry-After`.

---

## 4. Validating data with Pydantic

**Pydantic** turns type hints into run-time validation and parsing, the role Zod plays in
TypeScript (Book V, Chapter 4) and data annotations plus System.Text.Json play in .NET:

```python
# src/beacon_tools/schemas.py
from datetime import datetime
from typing import Literal

from pydantic import BaseModel, ConfigDict, Field, field_validator
from pydantic.alias_generators import to_camel


class ApiModel(BaseModel):
    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True, frozen=True, extra="ignore")


class TicketSummary(ApiModel):
    id: str = Field(pattern=r"^T-\d+$")
    title: str
    status: Literal["Open", "InProgress", "Resolved", "Closed"]
    priority: Literal["Low", "Normal", "High", "Urgent"]
    assignee: str | None
    created_at: datetime
    comment_count: int = Field(ge=0)

    @field_validator("created_at")
    @classmethod
    def must_be_timezone_aware(cls, v: datetime) -> datetime:
        if v.tzinfo is None:
            raise ValueError("created_at must include a UTC offset")
        return v


class Page[T](ApiModel):
    items: list[T]
    next_cursor: str | None = None
```

```python
ticket = TicketSummary.model_validate({"id": "T-42", "title": "VPN", "status": "Open", "priority": "High",
                                       "assignee": None, "createdAt": "2026-10-07T09:00:00Z", "commentCount": 2})
ticket.created_at          # datetime(2026, 10, 7, 9, 0, tzinfo=UTC)
ticket.model_dump(by_alias=True, mode="json")   # back to camelCase JSON
```

Invalid data raises a `ValidationError` listing every field problem with its location, the
same quality of error you'd want from an API (Book III, Chapter 5).

- `extra="ignore"` makes the client a **tolerant reader**: new API fields don't break it
  (Book III, Chapter 4).
- Pydantic models can also be generated from Beacon's OpenAPI document
  (`datamodel-code-generator`), keeping this third consumer of the contract in sync (Book VII,
  Chapter 2).

---

## 5. Working with files and data

### Paths, CSV and JSON

```python
import csv
import json
from pathlib import Path

out = Path("exports") / "open-tickets.csv"
out.parent.mkdir(parents=True, exist_ok=True)

with out.open("w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["id", "title", "priority", "assignee", "created_at"])
    writer.writeheader()
    for t in tickets:
        writer.writerow({"id": t.id, "title": t.title, "priority": t.priority,
                         "assignee": t.assignee or "", "created_at": t.created_at.isoformat()})

data = json.loads(Path("articles.json").read_text(encoding="utf-8"))
```

Always specify `encoding="utf-8"` (the default varies by platform) and `newline=""` for CSV.

> **⚠️ What can go wrong:** CSV files opened in Excel can execute formulas: a ticket title
> starting with `=`, `+`, `-` or `@` becomes a formula (**CSV injection**). When exporting
> user-provided text for spreadsheets, prefix such values with a single quote or escape them.

### Tabular data: pandas and Polars

For analysis and transformation of larger tabular datasets:

- **pandas**: the long-established DataFrame library; enormous ecosystem.
- **Polars**: a newer, much faster DataFrame library (Rust core, lazy queries, multi-threaded).

```python
import polars as pl

df = pl.read_csv("exports/open-tickets.csv", try_parse_dates=True)
summary = (
    df.group_by("assignee")
      .agg(pl.len().alias("open"), pl.col("created_at").min().alias("oldest"))
      .sort("open", descending=True)
)
print(summary)
```

That's Book IV's SQL aggregation (Chapter 2) or Book I's LINQ report, on a file. For data
already in PostgreSQL, run the SQL there instead; DataFrames are for data outside the
database, or for exploration.

---

## 6. Command-line tools

**Typer** builds CLIs from type-hinted functions (or use the standard library's `argparse`):

```python
# src/beacon_tools/cli.py
import asyncio
from pathlib import Path
from typing import Annotated

import typer
from rich.console import Console
from rich.table import Table

from .client import BeaconClient
from .export import write_csv

app = typer.Typer(help="Beacon support tools.")
console = Console(stderr=True)


@app.command()
def export(
    status: Annotated[str, typer.Option(help="Ticket status to export")] = "Open",
    out: Annotated[Path, typer.Option(help="Output CSV path")] = Path("tickets.csv"),
) -> None:
    """Export tickets to CSV."""
    async def run() -> int:
        async with BeaconClient.from_env() as client:
            tickets = [t async for t in client.list_tickets(status=status)]
        write_csv(tickets, out)
        return len(tickets)

    count = asyncio.run(run())
    console.print(f"[green]Exported {count} tickets to {out}[/green]")


@app.command()
def workload() -> None:
    """Show active and breaching tickets per agent."""
    ...  # uses models.workload_by_agent and prints a rich Table


if __name__ == "__main__":
    app()
```

```bash
uv run beacon-tools export --status Open --out open.csv
uv run beacon-tools --help
```

Good CLI behavior (the same rules as Book I, Chapter 8's .NET CLI):

- **Exit codes**: 0 for success, non-zero for failure (Typer raises `typer.Exit(code=1)`).
- **Output to stdout, diagnostics to stderr**, so output can be piped.
- **Helpful `--help`** generated from docstrings and annotations.
- **Dry-run modes** for anything destructive (`--dry-run` that prints what would change).
- **Idempotency** where possible: running an import twice shouldn't duplicate data (Book III,
  Chapter 4's idempotency keys work from scripts too).

---

## 7. Automation and scripting patterns

### Logging

```python
import logging

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s %(message)s")
log = logging.getLogger("beacon_tools.import")

log.info("Imported article", extra={"slug": slug})     # structured context (with a JSON formatter)
```

For anything that runs unattended (scheduled jobs, CI), log in JSON so the output is
queryable (Book III, Chapter 3).

### Concurrency with limits

Bulk operations against an API should be concurrent but bounded (Book V, Chapter 2's
`mapWithLimit`, in Python):

```python
import asyncio

async def import_articles(client: BeaconClient, articles: list[ArticleIn], limit: int = 5) -> None:
    semaphore = asyncio.Semaphore(limit)

    async def one(article: ArticleIn) -> None:
        async with semaphore:
            await client.upsert_article(article)       # idempotent: keyed by slug

    async with asyncio.TaskGroup() as tg:              # cancels the rest if one fails; raises an ExceptionGroup
        for a in articles:
            tg.create_task(one(a))
```

`TaskGroup` (Python 3.11+) is structured concurrency: no task outlives the block, and failures
aren't silently lost.

### Testing scripts

Scripts deserve tests too, especially those that change data. **respx** mocks httpx at the
transport level (MSW for Python; Book VI, Chapter 7):

```python
import respx
import httpx
import pytest
from beacon_tools.client import BeaconClient

@pytest.mark.asyncio
@respx.mock
async def test_list_tickets_follows_cursors():
    respx.get("https://beacon.test/api/tickets", params={"cursor": None}).mock(...)  # page 1 with nextCursor
    # ... page 2 without nextCursor
    async with BeaconClient("https://beacon.test", "token") as c:
        ids = [t.id async for t in c.list_tickets()]
    assert ids == ["T-1", "T-2", "T-3"]
```

---

## 8. Python for AI work: a preview

Book XI uses Python for:

- **Model SDKs**: the official Python SDKs from model providers (Anthropic, OpenAI, Azure
  OpenAI / Azure AI Foundry) and frameworks built on them.
- **Document processing**: extracting text from PDFs, Word documents and HTML; chunking for
  retrieval (Book XI, Chapter 6).
- **Embeddings and vector search** experiments with `pgvector` (Book XI, Chapter 5).
- **Evaluation**: running test sets of questions through Beacon's AI features, scoring answers,
  and tracking quality over time (Book XI, Chapter 8).
- **Notebooks** for exploring data and prompt behavior.

The production AI features themselves will live in Beacon.Api (C#), because they're part of
the product (Book XI, Chapter 10). Python is the lab and the test harness around them. That
division, Python for experimentation and data pipelines, the main stack for serving, is common
in real teams.

---

## 9. In practice: a knowledge-base import tool

Beacon's first customer wants their existing help center (a folder of Markdown files with
front matter) imported into Beacon's knowledge base. A complete, reliable tool:

```python
# src/beacon_tools/kb_import.py
from __future__ import annotations

import logging
import re
from pathlib import Path

import yaml
from pydantic import BaseModel, Field, ValidationError

log = logging.getLogger(__name__)
FRONT_MATTER = re.compile(r"^---\n(.*?)\n---\n(.*)$", re.DOTALL)


class ArticleIn(BaseModel):
    slug: str = Field(pattern=r"^[a-z0-9-]+$", max_length=120)
    title: str = Field(min_length=1, max_length=200)
    body: str = Field(min_length=1)
    tags: list[str] = []


def parse_article(path: Path) -> ArticleIn:
    text = path.read_text(encoding="utf-8")
    match = FRONT_MATTER.match(text)
    if not match:
        raise ValueError("missing front matter")
    meta = yaml.safe_load(match.group(1)) or {}                 # safe_load: never yaml.load on untrusted input
    return ArticleIn(slug=meta.get("slug") or path.stem.lower(),
                     title=meta.get("title", ""), body=match.group(2).strip(),
                     tags=meta.get("tags", []))


def load_folder(folder: Path) -> tuple[list[ArticleIn], list[tuple[Path, str]]]:
    ok: list[ArticleIn] = []
    failed: list[tuple[Path, str]] = []
    for path in sorted(folder.glob("**/*.md")):
        try:
            ok.append(parse_article(path))
        except (ValueError, ValidationError, yaml.YAMLError) as e:
            failed.append((path, str(e)))
    return ok, failed
```

The CLI command:

```python
@app.command("kb-import")
def kb_import(folder: Path, dry_run: bool = True, concurrency: int = 5) -> None:
    """Import Markdown articles into Beacon's knowledge base (dry run by default)."""
    articles, failed = load_folder(folder)
    for path, error in failed:
        console.print(f"[red]✗ {path}: {error}[/red]")
    console.print(f"{len(articles)} valid, {len(failed)} invalid")

    if dry_run:
        console.print("[yellow]Dry run: nothing imported. Use --no-dry-run to import.[/yellow]")
        raise typer.Exit(code=1 if failed else 0)

    async def run() -> None:
        async with BeaconClient.from_env() as client:
            await import_articles(client, articles, limit=concurrency)

    asyncio.run(run())
    raise typer.Exit(code=1 if failed else 0)
```

Properties: validates everything before changing anything, reports every invalid file,
**dry run by default**, idempotent upserts keyed by slug (rerunning is safe), bounded
concurrency to protect the API (Book IX, Chapter 7), `yaml.safe_load` (never `yaml.load` on
untrusted input, which can execute arbitrary code), and meaningful exit codes for use in
pipelines.

---

## 10. What can go wrong

- **Scripts without timeouts, retries or validation** that fail halfway through and leave
  inconsistent data.
- **Destructive scripts without dry runs** or confirmation.
- **Non-idempotent imports**, duplicating data on rerun.
- **Unbounded concurrency** overwhelming an API or database.
- **Unsafe deserialization**: `pickle` or `yaml.load` on untrusted data (remote code execution).
- **Encoding bugs** from missing `encoding="utf-8"`.
- **CSV injection** in exports opened in spreadsheets.
- **Secrets hard-coded** in scripts or notebooks committed to Git.

---

## 11. How an experienced engineer thinks about this

- **Tools are software**: typed, validated, tested, logged, reviewed.
- **Validate at the edge** with Pydantic; trust types inside.
- **Make scripts safe to run twice** and safe to run by someone else: dry runs, idempotency,
  clear errors, exit codes.
- **Use Python for its ecosystem** (data, AI, automation), and keep core product logic in the
  main stack.

---

## 12. Check yourself

**Questions**

1. Why set explicit timeouts on HTTP clients, and why reuse one client?
2. What does Pydantic do that type hints alone don't?
3. How does an async generator simplify paginated API access?
4. What makes a data-changing script safe to run in production?
5. What is CSV injection?
6. Why is `yaml.load` (or `pickle`) on untrusted data dangerous?
7. When would you use Polars or pandas rather than SQL?

**Exercises**

1. Implement `BeaconClient.list_tickets` and test pagination with respx.
2. Build the `export` command with CSV-injection protection, and test it with malicious titles.
3. Build `kb-import` with dry run and idempotent upserts; run it twice and verify no duplicates.
4. Use Polars to compute first-response-time percentiles per team from an export, and compare
   with the SQL from Book IV, Chapter 2.

**Interview-style questions**

- "How would you write a reliable data migration script?"
- "How do you validate external data in Python?"
- "When do you choose Python over C# for a task?"

---

## 13. Going deeper

- [HTTPX documentation](https://www.python-httpx.org/) and [Pydantic documentation](https://docs.pydantic.dev/)
- [Typer documentation](https://typer.tiangolo.com/)
- [Polars user guide](https://docs.pola.rs/)
- Al Sweigart, *Automate the Boring Stuff with Python* (free online) — for scripting ideas.

---

## Book X wrap-up

You can now read and write idiomatic Python, manage environments and dependencies
reproducibly with uv, and build reliable tools: typed API clients with pagination and
timeouts, Pydantic validation, safe CLIs with dry runs and exit codes, and data processing
with bounded concurrency. Beacon has a `beacon-tools` package with CI checks, ready for the
AI work ahead.

**Next:** [Book XI — AI Application Engineering](../11-ai/README.md).
