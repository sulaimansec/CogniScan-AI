# CogniScan AI

Claude-driven, non-destructive web app + LLM-widget vulnerability scanner.
**Only use against targets you are authorized to test.**

## Setup

```
py -3 -m pip install -r requirements.txt
py -3 -m playwright install chromium
```

Create a `.env` file (or set env vars directly):

```
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_MODEL=claude-sonnet-5   # optional, this is the default
```

## Run

```
py -3 cli.py scan --target https://your-authorized-target.com --confirm-scope
```

`--confirm-scope` is mandatory — the tool refuses to run without it. It is your
attestation that you have permission to test the target.

Useful flags:
- `--depth 3` crawl depth (default 2)
- `--no-ai-checks` skip Claude, run only the deterministic header checks
- `--max-concurrency`, `--rate-limit` throttle outbound requests (default 5 / 3 rps)
- `--allow-unsafe-methods` opt in to PUT/DELETE/PATCH probing (off by default)
- `--output-dir` where reports land (default `./cogniscan-reports`)

Reports: `cogniscan-reports/report.md` and `report.html` (latest run — overwritten each
scan). Every run is also archived to `cogniscan-reports/history/*.json`, so past scans
survive and can be diffed — see the History section in the dashboard.

## Web dashboard

```
py -3 -m streamlit run app.py
```

Same config as the CLI, plus: overall grade badge, one-click "Export All" (zip of
md+html+csv), and a **History & Before/After** panel — pick two past scans of the same
target (e.g. one before a fix, one after) to see what got resolved, what's new, and
what's still open.

## Deploying the dashboard

The scanner fires live requests at whatever URL you give it and burns your Anthropic
API key per scan — **do not deploy it to Streamlit Community Cloud**, that tier is public
with no auth in front, so anyone who finds the URL can point it at any site on your key.
Options, in order:
- **Self-host** ("Other platforms"): run it on your own VM behind a reverse proxy with
  basic auth or an SSO gate (Cloudflare Access, nginx `auth_basic`, Tailscale). This is
  the right fit for a single-operator pentest tool.
- **Local only**: `streamlit run app.py` on your machine — no deployment needed if you're
  the only user.
- Snowflake's offering is for enterprise data-stack integration, not relevant here.

## Self-check

```
py -3 test_smoke.py
```

## How it works

1. `recon.py` — Playwright crawls same-origin pages up to `--depth`, collecting
   forms/params/scripts/cookies/response headers, and fingerprints embedded
   chat/AI widgets (Intercom, Drift, custom `gpt`/`assistant`-named bundles, etc).
2. `ai_brain.py` — sends the recon summary to Claude, which proposes attack
   hypotheses (OWASP Top 10 + business logic + OWASP LLM Top 10 when a chat
   widget was found), generates non-destructive payloads per hypothesis, and
   analyzes each probe's response to render a verdict + severity + CVSS.
3. `scanner.py` — fires the payloads over `httpx` with concurrency + rate
   limiting, restricted to GET/POST/HEAD/OPTIONS unless unsafe methods are
   explicitly allowed. Also runs cheap deterministic checks (missing security
   headers) without any model call.
4. `reporter.py` — renders findings into Markdown and a self-contained HTML
   report: posture summary, weaknesses, verified vulnerabilities w/ PoC, and
   remediation steps.

## Safety notes

- Every Claude prompt is pinned with rules against destructive/DoS/exfiltration
  payloads and against escalating to unsafe HTTP methods.
- The scanner enforces the method allowlist independently of what the model
  returns — a model mistake can't send a live DELETE.
- LLM-specific tests only *probe and read* chatbot behavior (prompt injection,
  system-prompt leakage) — they don't instruct any live system to act.
