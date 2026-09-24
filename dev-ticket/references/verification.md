# Verification — per-repo checks before opening a PR

Goal: run the repo's **real** checks and capture pass/fail evidence for the PR status header.
Run all commands **inside the worktree** (`<BASE>\.dev-worktrees\<slug>\{{PROJECT_KEY}}-####`), never the main clone.

## The authoritative source: the repo's CI config
Before guessing commands, read the repo's CI config. It lists exactly what CI runs (install, build, lint, test steps) — **mirror those commands**. They are the contract the PR must pass. The per-stack defaults below are the fallback when no pipeline file exists or a step is unclear.

Also check for a `Makefile`, `composer.json`/`package.json` `scripts`, or `README` "development"/"testing" section — teams often wrap the real commands there.

## Getting a runnable toolchain in the worktree
A fresh worktree has no `node_modules` / `vendor` / Gradle caches, so checks can't run until deps exist. Two options:
- **Reuse the main clone's deps (fast).** If the worktree's lockfile is **identical** to the main clone's (`diff` the `package-lock.json` / `composer.lock`), junction the deps in rather than installing: on Windows, `New-Item -ItemType Junction -Path "<worktree>\node_modules" -Target "<main clone>\node_modules"`. Confirm the locks match first — mismatched locks → install instead. **This junction must be removed safely at cleanup** (SKILL Phase 10): `rmdir` the link, never `Remove-Item -Recurse` through it, or you delete the shared target.
- **Fresh install (correct, slower).** `npm ci` / `composer install` in the worktree.

If neither works (no registry access, private deps fail), mark the repo **Unverified** and say the toolchain couldn't be set up — CI will run the checks.

## Detect the package manager / build tool first
- Node: lockfile decides — `pnpm-lock.yaml` → `pnpm`, `yarn.lock` → `yarn`, `package-lock.json` → `npm`. Run install before anything (`<pm> install` / `--frozen-lockfile` in CI style).
- PHP/Symfony: `composer.json` present → `composer install`. Tests via `bin/phpunit` or `vendor/bin/phpunit`.
- Kotlin: `gradlew`/`gradlew.bat` present → use the wrapper.

## Per-stack defaults (fallback when the pipeline file doesn't specify)

### React / TypeScript / Vite
- install: `<pm> install`
- typecheck/build: `<pm> run build` (vite build usually runs `tsc` first)
- lint: `<pm> run lint` (eslint)
- test: `<pm> run test` (vitest/jest) — use `--run`/`--watch=false` so it doesn't hang
- If a script is missing, run the tool directly (`npx tsc --noEmit`, `npx eslint .`).

### NestJS / TypeScript
- install: `<pm> install`
- build: `<pm> run build` (nest build / tsc)
- lint: `<pm> run lint`
- unit test: `<pm> run test` (jest); e2e (`<pm> run test:e2e`) only if it can run without live infra — skip and note if it needs a DB/the event bus.

### Symfony / PHP
- install: `composer install --no-interaction`
- tests: `vendor/bin/phpunit` (or `bin/phpunit`)
- static analysis: `vendor/bin/phpstan analyse` if configured (`phpstan.neon`/`.dist`)
- style: `vendor/bin/php-cs-fixer fix --dry-run --diff` if configured (`.php-cs-fixer.dist.php`)
- Many of these run via `docker compose` / `make` in dev — if the pipeline uses a Docker step and Docker isn't available locally, mark **Unverified** and say so; do not claim green.

### Kotlin / Gradle
- build + test: `./gradlew build` (compiles, runs tests). On Windows use `gradlew.bat`.
- lint: `./gradlew ktlintCheck` or `./gradlew detekt` if configured.
- Needs JDK 23 on PATH — if absent, **Unverified**.

## Verifying promotion branches (staging / main)
The change is implemented and verified once on the development branch (SKILL Phase 5). When it is promoted to `staging`/`main` (SKILL Phase 6), re-verification depends on how it applied:
- **Touched file byte-identical to the primary's** and the primary built clean → the build carries over; say "carries over from develop" rather than re-running.
- **Hand-ported** because the branch had drifted → the surrounding code differs, so **re-run the build** on that worktree. Confirm the diffstat matches the primary's and the behaviour is identical before opening the PR.
The `main` PR header is `⚠️` regardless (production deploy).

## Recording the result
For each repo capture a one-liner the PR header/body can quote, e.g.:
- `build ✓ · lint ✓ · jest ✓ (142 passed)`
- `build ✓ · lint ✓ · phpunit ✗ (2 failing: FooTest) → DRAFT, flagged`
- `Unverified — JDK 23 not available locally; CI will run gradle build`

Rules:
- A failed or unrunnable check → repo is **`⚠️ Unverified`** in the PR header. Still open the draft (so the human sees the work) with the failure named.
- Never paste long logs into the PR — quote the failing test names / first error line; attach detail only if short.
- Don't disable, skip, or weaken tests to get green. If a test is genuinely wrong per the ticket, fix it and call that out in the diff + PR body.
