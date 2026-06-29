# Technical Context

Engineering-practice contract for code-writing agents in this project. Every implementation must respect what's written here. If a constraint conflicts with the user's request, flag the conflict — don't silently bypass it.

## Overriding Principles
1. **Secrets live in Windows Credential Manager (via DPAPI / a `keyring`-equivalent), never in source, `.env`, or `app.config`.** No keyed service is wired in yet (Open-Meteo is keyless), but this rule applies the moment one is added.
2. **No breaking change to a public contract without an ADR in `docs/adr/`.** A version pin or a documented decision must back any incompatible change.
3. **The UI never crashes on a failed or slow weather fetch.** Degrade gracefully — show last-known data plus a visible staleness/error state — never surface an unhandled exception to the user.
4. **All network I/O is async and cancellable.** No blocking calls on the UI thread; every fetch is awaitable and accepts a `CancellationToken`.

## Frameworks
- **.NET 10 (LTS)** — application runtime and SDK (current LTS as of this document's date; pin in the `*.csproj` `TargetFramework`).
- **WPF (Windows Presentation Foundation)** — desktop UI framework. Windows-only by design.
- **MVVM** — the UI architecture pattern (view ↔ view-model ↔ model), so view-models stay unit-testable without a UI thread.

## 3rd-party tech
- **Open-Meteo** — weather-data provider. Free, **keyless** for non-commercial use; queried over its JSON HTTP API. This is the project's primary external seam.

## Packages in use
**UI / MVVM:** `CommunityToolkit.Mvvm` (observable objects, relay commands)
**HTTP:** `System.Net.Http` `HttpClient` via `Microsoft.Extensions.Http` (`IHttpClientFactory`); JSON over `System.Text.Json` (`System.Net.Http.Json`)
**DI / composition:** `Microsoft.Extensions.DependencyInjection`, `Microsoft.Extensions.Hosting`
**Logging:** `Serilog`, `Serilog.Sinks.File`
**Testing:** `xUnit`, `FluentAssertions`, `WireMock.Net` (stub the Open-Meteo seam for recorded-replay tests)

Reach for these before pulling in a new dependency. A new package is a small decision that compounds — prefer the framework's built-ins (`System.Text.Json`, `HttpClient`) over a third option.

## Instrumentation
`Serilog` writing **structured logs to a rolling file** under `%LOCALAPPDATA%\SimpleDesktopWeatherApp\logs\`. No telemetry leaves the machine — nothing phones home.

- **Always logged:** every Open-Meteo request (URL + HTTP status + latency) and every failure/exception with its stack trace.
- **Levels:** `INFO` for a normal successful fetch · `WARNING` when falling back to stale/last-known data · `ERROR` for an unhandled or repeated failure.

## User Feedback Approach
- **Channel:** in-app only. Status and errors are surfaced inline in the window (status line / banner). No email, Slack, or toast notifications.
- **Degraded state is visible, not silent.** On a failed fetch, show last-known data with a clear message — e.g. *"Couldn't update — showing data from 14:32."* (ties to Overriding Principle #3).
- **Tone:** terse, plain, non-technical. Never show stack traces, HTTP status codes, or raw API responses to the user — those go to the log, not the UI.

## Branching and Environments
- **Model:** trunk-based. `main` is the single long-lived branch.
- **Branches:** human work on a short-lived branch → PR → `main`; AFK work on one `story/<issue#>-<slug>` branch per Story, squash-merged into `main` (see `CONTRIBUTING.md`).
- **Environments:** no hosted environments — this is a local desktop app. Two build flavours: a **local development build** (`dotnet run` / debug) and a **packaged release build** (the distributable Windows artifact).
- **Protected:** `main` — no direct pushes; every change lands via a PR that passes the `ci` check.

## Git Guardrails
- Never force-push to `main`, and never `git reset --hard` / `git clean` / delete a branch without explicit confirmation.
- Never commit secrets, credentials, or local config carrying them (`.env` and `*.local` are git-ignored; keep it that way).
- Never rewrite published history on a shared branch.
- Hook-enforced safety nets available but not yet installed: `/setup-pre-commit` (lint/format/test at commit time) and `/git-guardrails-claude-code` (block destructive git ops). Install when the codebase warrants it.

## Testing & the ratchet
> Seeded from the Factory standard — **opt-out, not opt-in**. Trim a line only with a deliberate, recorded decision; don't start from nothing. Full standard: <https://github.com/kitcox-dev/enate-claude-skills/blob/main/docs/testing-and-the-ratchet-standard.md> · the *why*: <https://github.com/kitcox-dev/enate-claude-skills/blob/main/docs/agentic-e2e-testing-principles.md>.
>
> **Project adaptation (recorded decision):** this app has no LLM. The standard's "LLM seam" maps here to the **Open-Meteo HTTP API seam**; "replayed LLM fixtures" become **replayed HTTP-response fixtures** (via `WireMock.Net`). The three-tier model and the ratchet are kept intact.

- **Test the contract, not the provider.** Assert on the deterministic envelope (parsed model shape, view-model state transition, fallback-on-failure behaviour, log/side-effect ordering), never on the exact wording of Open-Meteo's payload.
- **Every seam gets a real-IO test on at least one side.** Mock-on-both-sides only proves the mocks agree; defects live at integration seams (HTTP, file I/O), not inside view-models.
- **Three tiers, each catching a class the others can't.** Tier 1 — recorded-replay (real local file/UI-less I/O, the Open-Meteo seam faked with replayed HTTP fixtures; every commit); Tier 2 — live, disposable (real HTTP against Open-Meteo from a throwaway run; bounded cost, scheduled); Tier 3 — live (the packaged app against the real provider; continuous/manual smoke).
- **Platform matrix.** Filesystem/subprocess-touching code runs Tier 1 + Tier 2 on every Windows version the app supports.
- **Plan test coverage at the Feature level, not the Story level.** Each Feature declares at kickoff which tiers it needs, which fixtures/substrates must exist, and a Tier-2 cost/quota ceiling.
- **The ratchet — every Tier-3 defect pins a cheap-tier regression.** A live defect's fix lands with a Tier-1 (or Tier-2) regression that would have caught it, encoding the **class** of defect **bounded by the seam** where it surfaced — not the exact reproducer, and not a project-wide sweep. A Story that surfaced from a live defect carries that ratcheting fixture as an acceptance criterion.
- **Disposable substrate, not shared.** Tier-2 runs are self-contained and torn down; reproducibility comes from a seed (fixture manifest/script), never persistent state.
- **Assert on observable behaviour, not internal state.** For Tier 2/3 the target is what an operator could verify from external artefacts — rendered UI state, written log events, the file cache on disk.
