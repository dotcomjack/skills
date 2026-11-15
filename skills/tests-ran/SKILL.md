---
name: tests-ran
description: Runs a repo's real test suite and refuses to call a green run a pass until it can prove tests actually executed. Detects the runner per package (vitest, jest, node --test, pytest, xcodebuild), counts the tests that ran, and treats zero collected as a FAILURE. Use when the user says "/tests-ran", "run tests", "run the tests", "test this", "run the suite", "do the tests pass", "are tests green", "did I break anything", "check if tests pass", or names a repo or file to test.
argument-hint: "[repo-path] [test-file-or-pattern]"
---

## tests-ran: prove the suite RAN, then prove it PASSED

Two claims, and the first one is the one that gets faked. A runner that collected zero tests, a package with no `test` script, a monorepo root with no `package.json`, and a missing coverage plugin all produce output that reads like a result. This skill measures the number of tests that actually executed and gates on it. Anything else is a report, not a verdict.

### Honesty rules (non-negotiable)

- **Skipped is not executed.** Every runner's headline count includes skipped tests.
  Verified on a real machine: three `test.skip` cases produce `tests 3 / pass 0 / skipped 3`
  and a gate reading `tests` calls that three executed. Every counter below uses
  **pass + fail** and excludes skipped and todo, so `RAN` means genuinely ran.
- **Zero tests executed is a FAILURE.** Not "all tests passed", not "nothing to run", not "clean". This is the single reason this skill exists.
- **Exit codes are not interchangeable, and several runners lie.** Verified on a real machine: `node --test` in a directory with no test files prints `tests 0 / pass 0 / fail 0` and **exits 0**. `vitest run --passWithNoTests` reports `"success": true` with `"numTotalTests": 0` and **exits 0**. `pytest` exits **5** when nothing was collected, which is neither 0 nor a normal failure code. `npm test` with no `test` script exits **1** with `Missing script: "test"`, indistinguishable by exit code from a real test failure. Never gate on the runner's exit code alone.
- **`grep` exits 0 when it FINDS a match.** In Phase 1 a grep hit is BAD news, so **exit 0 there is the failing outcome and exit 1 is clean**. Read each gate's stated required outcome instead of assuming.
- **Evidence or it did not happen.** Every claim carries the command and its real summary line.
- **Say so plainly when you could not verify.** "The iOS suite did not run because no simulator was available" is a complete answer. "Tests pass" without a count is not.

### Phase 0: resolve the target, then GATE 0 (this gate can fail)

Read the repo's own `CLAUDE.md` (or `AGENTS.md`) first, and work in the canonical clone, never a synced mirror or an archived copy.

A real repo often holds several independent suites in subdirectories with no root `package.json`, so enumerate before running anything. `git ls-files` is used deliberately: a plain `find` picks up build output and vendored checkouts (verified: it surfaced `.next/standalone`, `.open-next`, and `ios/build/SourcePackages` as fake suites).

```sh
REPO=<repo-root>
python3 - "$REPO" <<'PY'
import json, os, subprocess, sys
root = sys.argv[1]
files = subprocess.run(["git", "-C", root, "ls-files"],
                       capture_output=True, text=True).stdout.split("\n")
found = 0
for f in files:
    if "_archive/" in f or "/node_modules/" in f:
        continue
    base, dirn = os.path.basename(f), os.path.dirname(f) or "."
    if base == "package.json":
        try:
            s = (json.load(open(os.path.join(root, f))).get("scripts") or {}).get("test")
        except Exception:
            continue
        runner = "none"
        for k in ("vitest", "jest", "node --test", "mocha", "ava"):
            if s and k in s:
                runner = k
                break
        else:
            runner = "custom" if s else "none"
        print(f"{'NODE':6} {dirn:30} {runner:11} {(s or 'NO test script')[:60]}")
        found += bool(s)
    elif base in ("pyproject.toml", "pytest.ini", "tox.ini"):
        print(f"{'PYTHON':6} {dirn:30} {'pytest':11} python3 -m pytest")
        found += 1
    elif base == "go.mod":
        print(f"{'GO':6} {dirn:30} {'go test':11} go test ./...")
        found += 1
    elif f.endswith(".xcodeproj/project.pbxproj"):
        print(f"{'XCODE':6} {f[:-16]:30} {'xcodebuild':11} xcodebuild test")
        found += 1
print(f"\nrunnable suites found: {found}")
sys.exit(0 if found else 1)
PY
echo "DETECT EXIT=$?"
```

**Required outcome: `DETECT EXIT=0` and at least one suite listed.** Verified both directions on real repos: a store monorepo prints 3 suites and exits 0, a folder of static dashboards prints `runnable suites found: 0` and exits 1.

`DETECT EXIT=1` means STOP and report "this repo has no test suite". Do not scaffold one uninvited and do not report a pass. If the user passed a file or pattern argument, pick the one suite that owns that path and run only that suite.

Two shapes this gate catches before they waste a run:

- **A monorepo root with no `package.json`.** Verified: `npm test` at the root of a store monorepo exits **254** with `Could not read package.json`. The real suites are in `backend` and `web`.
- **A package with a `package.json` but no `test` script.** Verified: `npm test` in a store admin package exits **1** with `Missing script: "test"`. That is not a test failure and must never be reported as one.

### Phase 1: sweep for the escape hatches that manufacture a green run

Two greps, before running anything. **A hit here is bad news.**

```sh
cd "$REPO"
git ls-files | grep -E '(vitest|jest)\.config\.|package\.json$' | xargs grep -n 'passWithNoTests'
echo "grep exit=$?"
```

**Required outcome: exit 1 and no output.** `grep` exits 0 when it finds a match, so **exit 0 here is the FAILING outcome**: the repo has opted into "zero tests is fine", which is exactly what this skill exists to catch. Verified both directions: the flag is absent across every repo checked (exit 1), and a file containing `passWithNoTests: true` makes the same grep print the line and exit 0. If it is present, report it as the run's first finding and run the suite without it.

```sh
git ls-files | grep -E '(^|/)package\.json$' | xargs grep -n '"test":.*\.test\.' 
echo "grep exit=$?"
```

**Required outcome: exit 1.** Exit 0 means a `test` script names its test files by hand, so a newly added test is silently never run. Verified on a node app: its `test` script lists 56 files by name while 58 are on disk, so two test files have never executed. Report the orphans by name (Phase 4 counts them) and offer to switch the script to a glob.

### Phase 2: run the suite and capture a machine readable report

Run from the suite's own directory, never the repo root. Each command below has been executed on a production repo.

**Clear the report artifacts FIRST, in the same shell call as the runner.** Every branch below writes to a fixed path, and nothing else ever removes them, so a runner that fails to execute at all leaves the PREVIOUS run's report in place and Phase 3 reads it and prints `GATE PASS: N tests executed`. That is the exact false green this skill exists to prevent, and it is live today: the pytest branch invokes `.venv/bin/python`, which does not exist in that Python worker, so the shell prints "no such file or directory", nothing runs, and the stale XML still yields a pass. Prefix every runner command with:

```sh
rm -f /tmp/tests-ran-report.json /tmp/tests-ran-report.xml /tmp/tests-ran-run.log
```

If the runner then fails to produce its report, `RAN` resolves empty or the parse errors out, and Phase 3 fails as it should instead of certifying a previous run.

**vitest** (a storefront app, a store backend, and a production web app):

```sh
npx vitest run --reporter=default --reporter=json --outputFile=/tmp/tests-ran-report.json
RAN=$(python3 -c "import json;d=json.load(open('/tmp/tests-ran-report.json'));print(d['numPassedTests']+d['numFailedTests'])")
FAILED=$(python3 -c "import json;print(json.load(open('/tmp/tests-ran-report.json'))['numFailedTests'])")
```

The dual reporter is deliberate: you get the human summary on screen AND the counts on disk. Verified it writes the JSON on a failing run too, not only a passing one.

**jest** (a React Native app):

```sh
npx jest --json --outputFile=/tmp/tests-ran-report.json
RAN=$(python3 -c "import json;d=json.load(open('/tmp/tests-ran-report.json'));print(d['numPassedTests']+d['numFailedTests'])")
FAILED=$(python3 -c "import json;print(json.load(open('/tmp/tests-ran-report.json'))['numFailedTests'])")
```

**node --test** (a node app). The counters are the only trustworthy signal here, because the exit code is 0 even when nothing ran:

```sh
npm test > /tmp/tests-ran-run.log 2>&1
RAN=$(awk '/ pass [0-9]+$/{p=$NF} / fail [0-9]+$/{f=$NF} END{print (p+0)+(f+0)}' /tmp/tests-ran-run.log)
FAILED=$(awk '/ fail [0-9]+$/{n=$NF} END{print n+0}' /tmp/tests-ran-run.log)
```

**pytest** (a Python worker). Use the repo's own virtualenv interpreter, not the system `python3`:

```sh
.venv/bin/python -m pytest -q --junitxml=/tmp/tests-ran-report.xml
RAN=$(python3 -c "import xml.etree.ElementTree as E;r=E.parse('/tmp/tests-ran-report.xml').getroot();s=r if r.tag=='testsuite' else r.find('testsuite');print(int(s.get('tests') or 0)-int(s.get('skipped') or 0))")
FAILED=$(python3 -c "import xml.etree.ElementTree as E;r=E.parse('/tmp/tests-ran-report.xml').getroot();s=r if r.tag=='testsuite' else r.find('testsuite');print(int(s.get('failures'))+int(s.get('errors')))")
```

Counting `errors` as failures is load-bearing. A pytest collection error is an ImportError, not a test result, and pytest exits **2** rather than 1. Verified on that repo: the checked-in `.venv` has no editable install of the package, so a bare run reports 12 collection errors, while the identical run with `PYTHONPATH=src` reports `66 passed`. Report a collection error as a broken suite, never as a passing one.

**xcodebuild** (measured on a SwiftUI app with a unit and a UI target):

```sh
xcodebuild test -project <name>.xcodeproj -scheme <Scheme> \
  -destination "platform=iOS Simulator,id=<UDID>" \
  -only-testing:<UnitTestTarget> -resultBundlePath /tmp/tests-ran.xcresult
xcrun xcresulttool get test-results summary --path /tmp/tests-ran.xcresult > /tmp/tests-ran-report.json
RAN=$(python3 -c "import json;d=json.load(open('/tmp/tests-ran-report.json'));print(d['passedTests']+d['failedTests'])")
FAILED=$(python3 -c "import json;print(json.load(open('/tmp/tests-ran-report.json'))['failedTests'])")
```

Three iOS gotchas, all hit on a real run:

- **Address the simulator by `id=<UDID>`, never `name=`.** Two simulators named `iPhone 17 Pro` exist here on different iOS versions, and `name=iPhone 17 Pro` hung the run for over 20 minutes with no error. Get the UDID from `xcrun simctl list devices available`.
- **Use `-only-testing:` to exclude the UI test target** unless the user asked for UI tests. Restricted to the unit target, that suite finished in 25 seconds and reported `Executed 154 tests, with 9 tests skipped and 0 failures`.
- **A scheme with no test target fails loudly, which is correct.** Verified on an iOS app: `xcodebuild test` exits **66** with `Scheme <AppName> is not currently configured for the test action.` Report that as "no iOS suite exists", not as a failure of the code.

**Any other runner:** run its documented command, capture the output, and extract `RAN` and `FAILED` from its own summary line. If you cannot extract a count, say so and report the run as unverified rather than passing.

### Phase 3: GATE 1, the count gate (the primary job of this skill)

Run this after every suite, with that suite's `RAN` and `FAILED`.

```sh
python3 - "$RAN" "$FAILED" <<'PY'
import sys
ran, failed = int(sys.argv[1]), int(sys.argv[2])
if ran == 0:
    print("GATE FAIL: zero tests executed. That is a FAILURE, not a pass.")
    sys.exit(1)
if failed:
    print(f"GATE FAIL: {failed} of {ran} tests failed")
    sys.exit(1)
print(f"GATE PASS: {ran} tests executed, 0 failed")
PY
echo "GATE EXIT=$?"
```

**Required outcome: `GATE EXIT=0` and a `GATE PASS` line naming a non-zero count.** Verified on real runs in both directions:

| Suite | Runner | Gate output |
|---|---|---|
| a storefront app | vitest | `GATE PASS: 359 tests executed, 0 failed` (exit 0) |
| a production web app | vitest | 847 passed across 73 files, gate passes |
| a store backend | vitest | 899 passed across 72 files, gate passes |
| a node app | node --test | `RAN=528 FAILED=0`, gate passes |
| a SwiftUI app | xcodebuild | `GATE PASS: 154 tests executed, 0 failed` (exit 0) |
| a Python worker | pytest | `GATE PASS: 66 tests executed, 0 failed` (exit 0) |
| a React Native app | jest | `GATE FAIL: 3 of 38 tests failed` (exit 1) |
| empty directory | node --test | runner exits **0**, gate prints `GATE FAIL: zero tests executed` (exit 1) |
| pytest on an empty directory | pytest | runner exits **5**, gate prints `GATE FAIL: zero tests executed` (exit 1) |

That last pair is the whole point. The runner said fine; the gate said no.

`GATE FAIL` on a zero count is never resolved by re-running with `--passWithNoTests` or by pointing at a different path until something matches. Find out why nothing was collected: wrong working directory, a broken import, an unbuilt package, a config `include` glob that no longer matches the tree.

### Phase 4: GATE 2, test files that exist but never run

Passing 359 tests means nothing if 20 test files were never loaded. For a vitest or jest report, pass the **suite's own directory**, not the repo root:

```sh
SUITE=<the directory the runner ran in>
python3 - "$SUITE" /tmp/tests-ran-report.json <<'PY'
import json, os, re, subprocess, sys
root, report = os.path.abspath(sys.argv[1]), sys.argv[2]
rep = json.load(open(report))
ran = {os.path.relpath(r["name"], root) for r in rep.get("testResults", [])}
tracked = {f for f in subprocess.run(["git", "-C", root, "ls-files"],
                                     capture_output=True, text=True).stdout.split("\n")
           if re.search(r"\.(test|spec)\.[cm]?[jt]sx?$", f) and "_archive/" not in f}
orphans = sorted(tracked - ran)
print(f"tracked: {len(tracked)}   loaded by the run: {len(ran)}   never run: {len(orphans)}")
for o in orphans:
    print("  NEVER RUN:", o)
sys.exit(1 if orphans else 0)
PY
echo "ORPHAN GATE EXIT=$?"
```

**Required outcome: exit 0 with `never run: 0`.** Verified passing on a storefront app (25 tracked, 25 loaded), on a production web app (73 tracked, 73 loaded), and on a store backend (72 tracked, 72 loaded).

**Scoping this to the suite is load-bearing, and `git -C <suite-dir> ls-files` already returns paths relative to that directory.** Verified: the same gate pointed at the store monorepo root instead of `backend` reports 4 false orphans, because those four files belong to the sibling `web` suite that this run was never meant to load. In a monorepo, run the gate once per suite, then reconcile at the repo level: a test file that no suite claimed is a real orphan, one that a later suite loads is not.

For a hand-listed `test` script the same question is answered against the script instead of a JSON report; the Phase 1 second grep already flags that shape, and a node app is the live example with 2 orphans. Report orphans as a finding even when every executed test passed.

### Phase 5: typecheck, reported as a separate signal

A type error is not a test failure and must be bucketed separately.

```sh
npm run typecheck 2>/dev/null || npx tsc --noEmit
echo "TYPECHECK EXIT=$?"
```

**Required outcome: exit 0.** Verified clean on a storefront app (via its `typecheck` script) and on a production web app (via `npx tsc --noEmit`). Not every repo has a `typecheck` script, hence the fallback. A repo with no `tsconfig.json` skips this step and the report says "not run", never "passed".

### Coverage: only claim it when it exists

**Do not promise coverage by default.** Verified: none of the vitest repos it was measured on has a coverage provider installed, and `npm run test:coverage` on a production web app exits 1 with `MISSING DEPENDENCY Cannot find dependency '@vitest/coverage-v8'`. That output is a tooling gap, not a failing suite, and reporting it as a test failure is a lie in the other direction.

Check first, and only then run it:

```sh
ls node_modules/@vitest/coverage-v8 >/dev/null 2>&1 && npx vitest run --coverage \
  || echo "coverage provider not installed, coverage NOT MEASURED"
```

If the user asked for coverage and the provider is missing, say so in one line and offer to install it. Do not install a dependency into their repo uninvited.

### Phase 6: report

Lead with the number. Four buckets, worst first, each entry carrying the command that produced it.

1. **FAILING**: any `GATE FAIL`, including a zero count. Name each failing test, its file, and expected against actual. Hand root-causing to a separate debugging pass.
2. **BROKEN SUITE**: collection errors, a missing runner, a missing `test` script, a scheme with no test target. Distinct from a failing test, and say which one it is.
3. **NEVER RUN**: orphan test files from Phase 4, plus any suite this run did not execute and why.
4. **NOT RUN**: typecheck, coverage, or a whole suite that was skipped, each with its reason.

Close with one verification line naming every suite and its count, for example: `backend 899 passed (vitest), web 358 passed 1 skipped (vitest), ios 154 executed 0 failed (xcodebuild), typecheck clean, coverage NOT MEASURED (no provider installed).`

**"All tests passed" is earned only by a non-zero `RAN`, a zero `FAILED`, and a clean orphan gate on every suite the repo has.** If any of those did not run, the sentence is "N tests passed in <suite>, and <the rest> was not verified because <reason>".

### What this skill does NOT do

- It does not write, scaffold, or repair tests unless the user asks. A repo with no suite is reported, not filled in.
- It does not install dependencies, create a virtualenv, or edit a `package.json` to make a run succeed.
- It does not root-cause a failure, run a pre-commit rule sweep, or drive the live site in a browser.
- It does not commit, push, or deploy. A green suite is not a go-live, and the "ship it" gate still stands.
- It does not touch production, any live database, or any deploy target. Suites that need a live backend are reported as unverified rather than pointed at prod.
- It does not retry a zero-count run with `--passWithNoTests` or any other flag that turns the gate off.

---

From the operating rules dotcomjack.com is built on: one founder, many Claude Code sessions, a few dozen products. Each rule above names the failure that produced it. https://dotcomjack.com/skills#kit
