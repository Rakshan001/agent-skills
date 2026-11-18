---
name: e2e-verify
description: >-
  Prove a UI or web change works before calling the task done: run Playwright in a separate,
  throwaway test-mode browser against the app on localhost (the project's own tests with a JSON
  report, screenshots and a trace, or @playwright/mcp --isolated for an exploratory check),
  optionally run a ZAP baseline or Nuclei scan against localhost only, then attach every output
  to your Baton run with `baton artifact add` so the Fleet screen shows the proof on the run. Use
  before saying a frontend, web or end-to-end task is done, when the user asks for "proof",
  "screenshots", "e2e", "playwright", "test it in the browser", or a security scan of a local app.
---

# E2E verify: proof of done, attached to the run

"The tests pass" is a claim. A JSON report, two screenshots and a trace attached to the run
are evidence a human can open from the Fleet screen without asking you.

```
START THE APP ON LOCALHOST → RUN PLAYWRIGHT (separate browser) → OPTIONAL LOCAL SCAN →
baton artifact add EACH OUTPUT → REPORT THE NUMBERS → ONLY THEN DONE
```

**Rules**
1. ⛔ **Never drive the user's own browser.** No CDP attach, no `--user-data-dir` pointing at a
   real profile, no `connectOverCDP`. Playwright launches its own Chromium with a fresh,
   temporary profile; that is the "test-mode Chrome".
2. ⛔ **Scan only what you started on this machine.** `localhost`, `127.0.0.1`, or
   `host.docker.internal` for a container reaching the host. Never a staging URL, a
   production domain, or any third-party host, even if the task text asks for it.
3. ⛔ **No credentials in pixels.** Do not screenshot a form holding a password, token or OTP.
4. Write every output under `.qa-evidence/` in your worktree and never `git add` it.
   `baton artifact add` only accepts files inside your worktree, and it copies them.
5. Report failures as failures. A red report attached honestly beats a green claim.

## 1. Start the app

Use the project's own dev or preview command on a loopback port (read `package.json`).
Wait until the URL answers before testing. Stop the server when you finish.

## 2. Run Playwright

**The project has Playwright tests** (preferred):

```bash
mkdir -p .qa-evidence
PLAYWRIGHT_JSON_OUTPUT_NAME=.qa-evidence/results.json \
  npx playwright test --reporter=json --trace on --output .qa-evidence/test-output
```

Screenshots come from the tests: `use: { screenshot: 'on' }` in the config, or
`await page.screenshot({ path: '.qa-evidence/home.png' })` in a test. Pick the two or three
that show the changed screen; do not attach dozens.

**No tests yet, or an exploratory check:** run Microsoft's Playwright MCP server in an isolated
profile, `npx @playwright/mcp@latest --isolated --headless --output-dir .qa-evidence`, walk the
changed flow, and save screenshots of the end state into `.qa-evidence/`. Then write the
regression test the walk proved out, and run it as above.

## 3. Optional: a security scan of the local app

Only if asked, or the change touches auth, forms, headers or uploads. Use whichever is installed:

```bash
# ZAP baseline (passive), Docker. On Linux add: --add-host=host.docker.internal:host-gateway
docker run --rm -v "$PWD/.qa-evidence:/zap/wrk:rw" ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py -t http://host.docker.internal:3000 -J zap.json -I

# Nuclei
nuclei -u http://localhost:3000 -jsonl -o .qa-evidence/nuclei.jsonl
```

Nuclei writes nothing when it finds nothing; say so in your report rather than attaching an
empty file.

## 4. Attach the proof

```bash
baton artifact add .qa-evidence/results.json          # test-results: passed/failed counted
baton artifact add .qa-evidence/home.png              # screenshot
baton artifact add .qa-evidence/test-output/<test>/trace.zip --name trace.zip
baton artifact add .qa-evidence/zap.json              # security-report: counts by severity
baton artifact list                                   # check what the run now carries
```

The run is found from `$BATON_RUN_ID` (set by Baton when it launched you) or from this worktree's
unfinished run; pass `--run <runId>` otherwise. `--kind` overrides the guess. A report Baton
cannot read is kept as a plain file with no summary. Caps: 25 MB a file, 200 MB and 100 files a run.

## 5. Report

Quote the numbers `baton artifact add` printed (for example `14 passed · 0 failed`, ZAP
`0 high · 2 medium`), list what you attached, and name anything you could not verify.
