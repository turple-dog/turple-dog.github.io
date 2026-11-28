# turple-dog.github.io

<!-- BEGIN turple-dog/engineering-standards: synced, edit it there, not here -->
## Shared standards

The house conventions below come first. Vercel's rules listed here apply
on top of them; where the two disagree, the house convention wins.

- No Vercel coding rules apply to this repo (no UI or React code).

### When reviewing a change

- Review the way `.claude/skills/vercel-code-review/SKILL.md` says (bugs and
  security first, be certain before flagging, name the input that breaks it),
  plus the points below. The PR bot and Workflow's agents do too.
- Name the rule ID and say what the rule is, for example C4 (host port equals container port), when a change breaks a convention below (E3).
- Treat as serious anything that would break the service or its data if merged:
  it won't start (bad compose, missing env var, port clash), behaviour is wrong,
  data is lost or an existing database stops working, a secret is committed or
  logged (S1, S3), HTML is unescaped or SQL is string-built (K6), a service is
  exposed beyond LAN/VPN (N4), a network call has no timeout (K7), or a change to
  dns, reverse-proxy or vpn has no one-command rollback in the PR (R2).
- Check that the README (M2), .env.example (S1) and docker-compose.yml agree
  with any changed port, env var or URL.
- A major visual change (T2) needs before/after screenshots in the PR.
- Don't review `.claude/skills/`, this synced section (in CLAUDE.md or
  AGENTS.md), or lockfiles: they change only through engineering-standards
  or the package manager.


## turple-dog conventions

Every repo is one self-hosted service (or a stack of them) running with
`docker compose` on the `homelab` server (`192.168.1.201`). Services are
reached on the LAN or over the WireGuard VPN, never directly from the internet.

Most of these rules describe how the existing repos already work. Sections
that say *Applies to* cover kinds of project that are still being set up;
they don't apply to repos outside that scope. Follow the rules when writing
code; check against them when reviewing. If a rule is wrong for a specific
case, keep the exception and say why in a comment next to it.

Rule IDs (`C1`, `S2`...) let reviewers cite a rule; a citation always says
what the rule is too (E3).

### Compose (`docker-compose.yml`)

- **C1** Service name, `container_name` and repo folder name are the same
  (`workouts` / `workouts` / `workouts/`).
- **C2** `restart: unless-stopped` on every long-running service.
- **C3** Every service runs in a container: own code ships a `Dockerfile`
  and compose builds it with `build: .`. Third-party images pin a version
  tag, not `latest`, unless a comment explains why it tracks latest.
- **C4** Ports: host port equals container port, overridable from `.env`:
  `"${WORKOUTS_PORT:-8770}:8770"`. A new port must not clash with any other
  repo's compose file — grep them all before picking one. New in-house apps
  use the 87xx–88xx range.
- **C5** State lives in bind mounts next to the compose file (`./data`,
  `./config`), never anonymous volumes. Every volume line has a comment
  saying what lives there and whether it needs backing up
  (`# SQLite database. Back this folder up.`).
- **C6** Run as non-root: `user: "1000:1000"` in compose or `USER` in the
  Dockerfile.
- **C7** Set `TZ=America/Vancouver` for anything that schedules, logs or
  displays times.
- **C8** Talking to another stack goes over its named external network
  (e.g. `media` for Jellyfin), with a comment saying who reaches whom. Host
  paths into another repo (`/home/bararda/repos/jellyfin/...`) are allowed
  only with a comment explaining the dependency.

### Secrets and config

- **S1** Never commit secrets. Real values go in `.env`, which is
  gitignored. `.env.example` is committed, lists every variable the compose
  file reads, with blank or dummy values and a comment per variable.
- **S2** Compose reads optional secrets as `${VAR:-}` so the stack still
  starts without them.
- **S3** Secrets never appear in logs, error messages, status pages, URLs
  or committed config files.
- **S4** Data directories (`data/`, databases, caches, backups, `logs/`) are
  gitignored, as are `__pycache__/`, `*.pyc`, `node_modules/` and build
  output (`dist/`).

### Dockerfiles

- **D1** Official slim base images pinned to a major version
  (`python:3.13-slim`, `node:24-bookworm-slim`).
- **D2** Deliberately unpinned dependencies get a comment saying why and how
  to update them (see `twitch-tuner/Dockerfile` for streamlink).
- **D3** `pip install --no-cache-dir`; Python images set `PYTHONUNBUFFERED=1`.
- **D4** `EXPOSE` the port, and set `PORT` / data paths through `ENV` so
  the code and the image agree.
- **D5** HTTP services get a `HEALTHCHECK` against a cheap `/healthz` route.
- **D6** `COPY` only the files the app needs, not the whole context.

### Code

- **K1** Fewer dependencies is better. Use the standard library first: Python
  `http.server`, `urllib`, `sqlite3`; Node `node:http`, `node:sqlite`,
  built-in `fetch`. A new dependency must be worth more than the code it
  saves, and the PR says why.
- **K2** Each entry-point file opens with a docstring or comment saying what
  the service does and how it fits with the rest (who calls it, what it
  calls).
- **K3** Configuration is read from env vars with defaults, as UPPER_CASE
  constants at the top of the file. Names carry units
  (`STATUS_TTL_SECONDS`, `GUIDE_HOURS`).
- **K4** Comments explain *why* (a workaround, a limit, a choice between
  options), not what the next line does. How much to write is K13.
- **K5** SQLite lives in the data directory with WAL and foreign keys on.
  Schema changes are additive (`CREATE TABLE IF NOT EXISTS`, `ALTER TABLE
  ADD COLUMN` guarded by a check), because the old database is always there.
  With an ORM (K16), schema changes are its migrations, and each one only
  adds.
- **K6** SQL always uses parameters, never string formatting. HTML output is
  escaped (`html.escape`, `quoteattr`, or `textContent` in the browser).
- **K7** Calls to outside services (APIs, scrapers, streamlink) have a
  timeout and fail soft: log, keep serving what's cached, and recover on the
  next cycle instead of crashing the service.
- **K8** Background loops and threads must not die silently. Catch and log
  per iteration.
- **K9** No vendor lock-in. Every outside vendor (cloud services, SaaS APIs,
  SDKs for payments, email, storage, auth, AI models, hosting) sits behind
  our own service interface named for what it does, not who provides it
  (`EmailSender`, `BlobStore`, `PaymentGateway`). App code calls only the
  interface. The vendor SDK is imported only in that interface's adapter,
  and the adapter is picked by config, so switching vendors means writing a
  new adapter and changing config, never changing callers. An open standard
  that several vendors implement (SQL, SMTP, the S3 API, OpenID Connect)
  can serve as the interface itself.
- **K10** Vendor types, IDs and errors don't leak past the adapter: map them
  to our own. Data is stored in formats we can export and move (SQL, JSON,
  files), not in features only one vendor has. Check for K9 and K10: a lint
  rule allows vendor SDK imports only in adapter folders (ESLint
  `no-restricted-imports`, Semgrep for Python and C#).
- **K11** File and folder names are kebab-case: lower-case words joined by
  hyphens, with dotted suffixes kept (`blade-frame.tsx`, `run-store.cs`,
  `side-nav.module.css`, `server/apps/host/host.csproj`). This
  covers C# and React files too; type and component names inside stay as the
  language has them. Python is the exception: `.py` files and Python package
  folders are `snake_case` (`run_store.py`), since `import run-store` isn't
  valid Python. Names a tool requires keep them (`README.md`, `CLAUDE.md`,
  `AGENTS.md`, `SKILL.md`, `Dockerfile`, `Caddyfile`, `Directory.Build.props`), and files
  copied from elsewhere keep theirs (`.claude/`, `vendor/`). Check: copy
  `checks/check-names.sh` from this repo into a repo's `scripts/` and run it
  from its checks; it fails on any name that doesn't fit. Godot repos get
  `scripts/check-names.sh` synced (it also allows `export_presets.cfg` and
  `AndroidManifest.xml`); a repo's own exceptions go in
  `scripts/check-names.conf` (`allow <name>`, `skip <path>`), not in a fork.
- **K12** Folder names spell out words instead of using acronyms or
  initialisms: `tests/` not `e2e/`, `pull-requests/` not `prs/`,
  `components/` not `ui/`. Folder names a tool requires (`.github/`) and
  product names (`github`) are fine. No script can tell an acronym from a
  word, so reviewers check this one.
- **K13** Exported code (functions, types, components and modules that
  other files or packages use) gets a full doc comment: what it does, its
  parameters and result, and how to use it. Code that isn't exported gets a
  minimal one: the context it's used in, plus a note on the implementation
  only if it's complex. Check: a lint rule requires doc comments on exports
  only (ESLint `jsdoc/require-jsdoc` with `publicOnly`, C# warning `CS1591`,
  Ruff `D1` for Python).
- **K14** Folder names don't repeat the repo's name or the folder they're in:
  `server/modules/agents/`, not `server/workflow-modules-agents/`. Namespaces
  and assembly names can carry it (`Workflow.Modules.Agents`). Check:
  `check-names.sh` fails on a folder whose name starts with the repo's.
- **K15** Server code is split by kind of project, so the entry points are
  plain from the tree: `apps/` holds one folder per process that runs (web
  host, worker, runner), each with its only entry point (`program.cs`, K2),
  wiring the rest together; `modules/` holds business logic, one folder per
  area (`modules/agents/`); `adapters/` holds everything that talks to
  something outside our code: vendor APIs (K9) and the database, including
  the ORM's provider and setup; `contracts/` holds the interfaces and types
  they meet through. Modules and adapters reference only `contracts/`, never
  each other. Check: a test reads the project references.
- **K16** C# reads and writes the database through EF Core, not hand-written
  SQL. Each module has its own `DbContext` over its own tables and uses only
  the provider-neutral EF Core API; the database adapter picks the provider
  and its settings (`UseSqlite`, K5). Raw SQL only where EF Core can't
  express it, parameterised (K6), with a comment saying why.
- **K17** C# names types explicitly: no `var`
  (`List<AgentRun> runs = await store.ListAsync(ct);`). Target-typed `new()`
  is fine, since the type is already on the left. Check: `.editorconfig`
  sets `csharp_style_var_* = false` with `IDE0008` as an error, and the
  build has `EnforceCodeStyleInBuild` on.
- **K18** No barrel files. An `index.ts` that only re-exports is allowed
  only as a package's entry point (`react-components/src/index.ts`);
  everything else is imported from the file that defines it. Check: ESLint
  `no-restricted-syntax` bans `export ... from` outside package entry files.
- **K19** Group by feature and keep the tree shallow. Code that's used
  together lives in one folder (`develop/pull-requests/` holds the list, the
  detail and the diff, not three sibling folders); only code shared across
  features sits on its own. A folder for one or two files doesn't go at the
  top of the repo: put it in the folder for its kind (`server/apps/browser/`).
- **K20** Log through a logging API with standard levels, never `print` or
  `console.log`: Python `logging`, C# `ILogger`, and in Node a small logger
  module over `console.error`/`warn`/`info`/`debug`. **error**: an operation
  failed and someone should look; include the exception and stack.
  **warning**: something went wrong but was handled, such as a fail-soft
  call (K7) or a rejected request. **info**: lifecycle and notable events,
  such as startup with port and config (no secrets, S3) or a finished job.
  **debug**: detail for troubleshooting, off by default. The level comes
  from `LOG_LEVEL` (default `info`, K3), logs go to stdout/stderr so
  `docker logs` shows them, and each line says what happened with the IDs
  needed to find it (`guide fetch for channel 42 timed out after 10s,
  serving cached`). Check: Ruff `T20` bans `print`, ESLint `no-console`
  outside the logger module, and `BannedApiAnalyzers` bans `Console.Write*`
  in C#.
- **K21** Log verbosely: someone with only `docker logs` should be able to
  tell what the service did and why it failed. Every caught exception,
  fallback, retry and rejected request logs at warning or error, never
  nothing. Startup logs the port and config in use at info, each background
  job logs at info when it runs and what it did (counts, duration), and each
  call to an outside service logs at debug what it called and how long it
  took. At debug, log the decisions the code makes and the inputs behind
  them, so `LOG_LEVEL=debug` explains a run. No script can tell a missing
  log line, so reviewers check this one.
- **K22** In an app agents act in (through MCP tools or an API they call), a
  state-changing operation is declared once, in one action list:
  a name, a description, typed input and result, its scope and a risk tier.
  The HTTP endpoint, the agent's tool, the command palette entry and the
  UI's button all come from that declaration, and every use is audited.
  Rules about an action (merge checks, who may do it) live in the action,
  never in its callers. Deliberate exceptions (sign-in, the vendor
  connection, an agent's own authority) are listed by name. Check: a server
  test fails when a state-changing endpoint exists outside the list.

### Web UI

- **U1** Phone-first. Check layouts at phone width; tap targets are big
  enough for a thumb.
- **U2** Never use `alert()`, `confirm()` or `prompt()`. Destructive actions
  confirm in the app, through the overlay U17 picks for a decision or an
  inline confirm; errors show as a toast or inline message.
- **U3** Single-file `index.html` (or a small `web/` build) served by the
  app itself is the norm. No separate web server container.
- **U12** An app's pages are folders in one app package with one build, not
  packages with their own `package.json` and bundler config. Packages are for
  code shared between apps (L2). Pages stay self-contained through lint
  (`no-restricted-imports`), not through separate builds.
- **U13** Display components take a data shape of their own, declared with
  the component, never a vendor's or an API's type: `Board` takes
  `BoardColumn[]` and `BoardCard`, not GitHub project items. The page maps
  its data to that shape, so another source (K9) needs a new mapping, not a
  new component.
- **U16** Body text runs 45 to 75 characters a line, 66 ideally: cap
  paragraphs with a typography token, not at the width of their container.
  Size the token by counting characters, not by `ch`, which is the width of
  a "0" and wider than the average letter: in Arial-like fonts `66ch` gives
  about 79 characters, and `56ch` about 65. Tables, code and labels aren't
  body text.
- **U17** Interrupt only when it's needed, and pick the overlay for the job:
  - *Modal dialog* (MUI `Dialog`): the person must acknowledge or decide:
    confirmations, short forms, critical choices. Escape and a backdrop click
    close it, except for a destructive decision. Keep it short; content that
    scrolls a lot belongs in a drawer or a page.
  - *Side drawer* (MUI `Drawer`, modal or persistent): detail panels,
    filters, long forms, so the page stays in view. Right for content and
    detail, left for navigation.
  - *Bottom sheet* (`Drawer anchor="bottom"`, peek or full height): actions,
    filters and secondary information on phones; rarely on desktop.
  - *Full-screen dialog*: complex forms and multi-step flows on phones, with
    close or back in a top corner. On desktop only for media and document
    viewers.

  Content the person may want beside the page is a drawer or an inline
  expansion, never a modal.
- **U18** Overlays change type with the viewport, not just size: a dialog at
  1440px can be a bottom sheet or a full-screen dialog at 375px. Every
  overlay can be dismissed, says where the person is (a title), and returns
  them to where they were: focus goes back to what opened it, and the page
  behind keeps its state.
- **U19** Navigation is persistent, layered and shallow. Primary navigation
  is always visible (one tap away on phones) and never moves, disappears or
  reorders. Layers: global navigation for top-level areas, local navigation
  for the pages in an area, in-page links for long pages; at most two
  navigation levels. The current place is always plain: active states, the
  page title, breadcrumbs, and `aria-current="page"`.
- **U20** Desktop and phone navigation differ. Desktop: the top bar is always
  visible, local navigation is a persistent or collapsible side nav, hover
  menus are fine, and navigation never hides behind a hamburger. Phone:
  global navigation goes in a hamburger or a bottom bar (3 to 5
  destinations), the side nav becomes a left drawer that closes when a page
  is picked, nothing depends on hover, intermediate pages are skipped where
  possible, and swipe can open a drawer but is never the only way.
  An app whose main way of working is an AI assistant may use a rail and a
  conversation instead of a top bar and side nav. Desktop: a rail of places
  that is always visible and never reorders (project switcher, Home, the
  inbox with its count, the stages of the work, activity, settings), the
  assistant's conversation always on screen, and a canvas where the work
  opens as tabs beside it, at one of three widths remembered per view: chat
  (conversation alone), split (conversation beside the canvas) or focus
  (canvas alone, composer docked at its foot). Phone: the conversation is
  the screen, with a bottom bar of 3 to 5 destinations, and a view opens
  full screen with Back and the composer docked above the bar. U19 applies
  unchanged (`aria-current="page"` on the rail's place, the view's title
  is the page's h1).
- **U21** A field's label sits above the field and is always shown: never
  inside it as a placeholder or a floating label that moves when the field is
  filled, and never on its border. A placeholder may show an example value,
  never stand in for the label. In MUI apps the design-system theme does this
  for every `TextField`, `Autocomplete` and `Select` (`MuiInputLabel` with
  `shrink` and a static position); fields don't override it.
- **U22** Choosing among things that already exist (repositories, projects,
  people, services) is a picker of them, never typed text such as a
  comma-separated list: checkboxes for a few, a searchable multi-select with
  the chosen ones as chips for many (MUI `Autocomplete multiple`, grouped
  where they group). Free text is only for what doesn't exist yet: a new
  name, a message.
- **U11** No emoji in UI, and no Unicode symbols standing in for icons
  (`✓`, `✕`). Use Lucide icons (free, ISC): `lucide-react` in React apps,
  the same icons as inline SVG in single-file apps, at one stroke width
  throughout an app (`strokeWidth={1.75}`). Apps still on Material Icons
  (`@mui/icons-material`) move to Lucide when they're next redesigned, not
  piecemeal, so one screen never mixes the two. Check: CI fails on
  `\p{Extended_Pictographic}` characters in UI files.

### Design system

*Applies to* apps built on the turple-dog design system. Existing single-file
apps keep following U3 until they're moved onto it.

- **U4** UI uses design-system components and tokens only. No raw colors,
  spacing, radii, font sizes, shadows or durations in app code.
- **U5** Use semantic tokens (`color.surface`, `space.md`), never primitive
  ones (`blue-500`), so every theme in the palette picker works without code
  changes.
- **U6** Tokens come from the design-system package for the platform (CSS
  variables, TS constants, Godot theme), never copied into the app, so an
  upgrade is a version bump.
- **U7** In the design system itself, every component has an example for
  each variant and state, and doc comments (props, usage, do and don't) that
  generate the hosted docs and their AI-readable copy.
- **U8** Web UI is React with MUI (Material UI). The MUI theme runs in
  CSS-variables mode (`cssVariables: true`), and CSS Modules use those
  variables (`var(--mui-palette-background-paper)`), not literal values.
- **U9** Style with `className` and CSS Modules, not the `sx` prop. Only
  generic components in the design-system UI package may use `sx`. Every
  component takes a `className` and puts it on its root, so a class can
  always override it; MUI's styles sit in a lower CSS layer
  (`enableCssLayer`) so that works without `!important`. Check: ESLint
  `no-restricted-syntax` bans `sx` outside the generic components, and a
  test renders each component with a `className` and checks its root has it.
- **U10** All MUI styling comes from one theme in the design-system package,
  built from its tokens. Apps don't call `createTheme` or add their own
  `ThemeProvider`, and a palette from the picker changes token values, not
  the theme. Keep `theme.components` overrides few, each with a comment
  saying why. Check: ESLint `no-restricted-imports` allows `createTheme` and
  `ThemeProvider` only in the theme module.
- **U14** Tables are data-driven: use the design system's `DataTable` with
  column definitions (key, header, how a cell renders) and rows. Don't build
  `TableHead`, `TableRow` and `TableCell` by hand. Check: ESLint
  `no-restricted-imports` allows MUI's table parts only in `DataTable`.
- **U15** Text in MUI apps uses `Typography` with a theme variant, not plain
  `h1`-`h6` or `p` tags: the variant picks the look and `component` the
  meaning (`<Typography variant="h6" component="h2">`). How text looks is
  changed in the typography tokens, not per component.
- **U23** A page or view renders the design system's `View` (title, status,
  actions, tabs) and `Page` (padding, reading width) in every state,
  loading and errors included, with the shell's controls passed in. It
  sets no outer padding, width cap or heading sizes of its own and draws no
  header of its own. Check: lint bans raw font sizes and literal
  breakpoints in CSS Modules; reviewers check the frame.

### Testing and verification

- **T1** Prove a change works before calling it done, cheapest check first:
  lint and typecheck, then the tests for the changed code, then the full
  suite, then the slow suites CI doesn't run (T4), stopping where the owner
  said to (T5). Keep the output short (quiet reporters, failures only) to
  save time and tokens.
- **T2** Major visual changes need before/after screenshots in the PR, at
  phone width (U1). Minor visual changes don't. A change is **major** if it:
  1. adds a new page or screen,
  2. changes a page's main data display to a different type of display
     (cards to a table), or
  3. affects all screens (global layout, navigation, theme or token changes).

  Replacing a page-specific component with a shared one that has the same
  type of display (a custom table becomes the design-system Table) is not
  major, even when it touches every screen.
- **T3** Test projects and suites live in the repo's top-level `tests/`,
  one folder per suite (`tests/server/`, `tests/playwright/`), not beside the
  code they test. A unit-test file that a JS test runner finds next to its
  source (`stack.test.ts`) can stay there.
- **T4** CI checks pull requests only: not a push to main (what's merged is
  what its PR checked), and a newer push cancels the older run
  (`concurrency` with `cancel-in-progress: true`). Check jobs on the homelab
  runner (T9) run on every PR, text-only ones included: each repo's `main`
  ruleset requires them, and a required check that never reports blocks the
  merge. Only jobs on GitHub-hosted runners (the macOS ones) and the Claude
  review skip text-only changes (`paths-ignore: ["**/*.md"]`), to save paid
  minutes and review usage. It runs the cheap checks
  (lint, typecheck, unit and server tests). Slow suites (end-to-end,
  browser) run only by hand (`workflow_dispatch`): whoever makes the change,
  an agent included, runs them before opening or updating the PR, unless
  the owner picked a lighter check (T5), and says in it what ran and how it
  went. Image builds on merge aren't checks and stay.
- **T5** On homelab work, ask the owner how to verify a change before
  verifying it, offering checks sized to the change (lint and the changed
  code's tests; run it and curl `/healthz`; the full T1 ladder; wait for
  CI). For a quick change, don't wait on the full CI/CD run or the slow
  suites: the owner's sweeper tasks catch what slips through. The PR still
  says what was checked (P1).
- **T6** An agent checking its own UI work keeps screenshots out of its main
  context: a subagent takes and reads them and replies with a text verdict.
  Capture only the element being checked (`locator.screenshot()` or `clip`),
  not the full page, at `deviceScaleFactor: 1`. Screenshots for the PR (T2)
  are for people and stay full-width.
- **T7** Waiting on CI never blocks the session. Poll in the background
  (`run_in_background`, Monitor) and keep working, not in a foreground
  `until gh pr checks ...; do sleep` loop.
- **T8** A bug fix comes with a test that fails without the fix and passes
  with it, in the same PR. If the bug can't be tested, the PR says why.
- **T9** Every workflow job runs on the org's self-hosted homelab runner
  (`runs-on: [self-hosted, homelab]`): checks, Android builds, release
  packaging, the Claude review and the standards sync. GitHub-hosted (paid)
  runners are only for jobs that need macOS (`runs-on: macos-15` for iOS and
  Xcode builds). Check: copy `checks/check-runners.sh` from this repo into a
  repo's `scripts/` and run it from its checks; it fails on any other
  `runs-on`.
- **T10** Screenshots shown to the owner for a decision (a contact sheet, a
  gallery, a set sent for approval) name each shot's language, and the
  screen where that isn't obvious (`Arabic · first-run card`). Leave
  pseudo-locale shots out, or label them `Pseudo-locale (test)`: they're
  layout checks, not translations.

### Shared packages

*Applies to* code reused across repos.

- **L1** Before writing a helper, check the shared packages. Code copied into
  a second repo gets moved into a package instead.
- **L2** Packages are small and do one job (`react-component`, `date-utils`,
  `data-validation`, `email`, `task-queue`, `authentication`). Only very
  generic code may cover more than one job.
- **L3** Packages are published on GitHub (GitHub Packages for npm and
  NuGet, GitHub Releases for Godot addons) and versioned with SemVer. Repos
  never depend on each other by path or git URL. Our own packages don't
  count as new dependencies under K1.
- **L4** Generic helpers (dates, grouping, formatting: `groupByDate`,
  `formatRelative`) go in the shared `utils` package, not in a page or a
  component. It's the one package that may hold more than one job (L2),
  as long as everything in it is generic.
- **L5** A package is published from its own repo when a release is
  published (Workflow's `publish_version`), with that repo's `GITHUB_TOKEN`
  and `packages: write`. The first publish creates the package; nothing else
  is set up by hand.
- **L6** Anything that installs our packages reads the token from
  `GITHUB_PACKAGES_TOKEN`, a classic token with `read:packages`: `.npmrc`
  has `//npm.pkg.github.com/:_authToken=${GITHUB_PACKAGES_TOKEN}`,
  `nuget.config` has `%GITHUB_PACKAGES_TOKEN%`, and an image build gets it
  as the compose build secret `github_packages_token`. Workflow's deployer,
  previews and agents already set it. CI sets it from the org secret:
  `GITHUB_PACKAGES_TOKEN: ${{ secrets.PACKAGES_READ_TOKEN }}`. Never install
  with `GITHUB_TOKEN`: it needs each package to grant each repo Actions
  access, by hand, with no API.

### Cloud apps

*Applies to* apps built to run in the cloud. Personal apps run on the homelab
and follow the compose and networking rules above instead.

- **A1** Containers keep no state of their own: sessions, files and queues
  live in external stores, and config comes from env vars (K3). This
  replaces C5 for cloud apps.
- **A2** Safe to scale out, scale in and scale to zero: start fast, shut down
  gracefully on SIGTERM, and make jobs safe to run twice.
- **A3** Data access goes through one storage layer, so hot and cold data can
  move between tiers without changing app code.
- **A4** The same image runs in every environment; only config differs.

### Godot

*Applies to* Godot projects: games and addon repos (a `project.godot`, or an
`addons/*/plugin.cfg`).

- **G1** The repo pins its Godot in `.godot-version` at the root: one line,
  `4.7.2` (meaning `4.7.2-stable`) or a pre-release such as `4.8-beta1`. It
  agrees with the version in `project.godot`'s `config/features`. Changing
  Godot version is a PR that changes this file.
- **G2** CI, check scripts and agents get Godot only through
  `scripts/godot.sh` (synced from here into every Godot repo), never a raw
  download: it fetches the pinned version, checks it against the release's
  SHA-512 sums and caches it. `GODOT=$(scripts/godot.sh --path)` for a
  script or `$GITHUB_ENV`, `scripts/godot.sh --templates` for export
  templates, and `scripts/godot.sh --headless --path . --import` to run it.
  Check: `scripts/check-godot.sh`, synced into every Godot repo from
  `checks/`, run from its checks; it fails without `.godot-version`,
  when that disagrees with `project.godot`, or when another file downloads
  a Godot release.

### Mobile apps

*Applies to* apps shipped to phones (iOS or Android), Godot games included.

- **H1** The app ID (iOS bundle identifier, Android application ID) is
  `org.turpledog.<app>`, the same on both platforms: lower case, the app's
  name with no separators (`org.turpledog.friendlycompetition`,
  `org.turpledog.brickbreaker`). Settle it before the first store upload;
  it can't change after that. Godot keeps it in `export_presets.cfg`,
  Capacitor and Expo in their app config; no script reads them all yet, so
  reviewers check this one.

### Making a service reachable

A new HTTP service needs all four of these, in the same PR or in linked PRs:

- **N1** A pair of blocks in `reverse-proxy/Caddyfile`:
  `<name>.home.emanthony.party` with `import cf_tls`, and `<name>.homelab`
  with `import internal_tls`, both `reverse_proxy host.docker.internal:<port>`.
- **N2** An AdGuard DNS rewrite in `dns/` for the new hostname.
- **N3** An entry in `status-dashboard/config/services.yaml` with
  `server: my-docker` and `container: <name>`, linking the
  `https://<name>.home.emanthony.party` URL.
- **N4** No port is published to the internet and no router port-forward is
  added. Outside access is through the VPN.

### High-risk repos: `dns`, `reverse-proxy`, `vpn`

- **R1** These sit on the network path the owner (and Claude Remote Control)
  uses to reach the server. A bad change can cut off the access needed to
  fix it.
- **R2** PRs touching them must say how to roll back in one command, and
  must never be auto-merged or auto-deployed.
- **R3** Don't restart or recreate these containers as part of a change
  without first asking whether the current session is connected through
  them. The `vpn` sidecars (`network_mode: service:wg-easy`) restart
  together.
- **R4** Git history is the backup. Don't add `*.pre-<thing>` copies of
  config files.

### READMEs

- **M1** Every repo has a `README.md`: one paragraph on what it is, a short
  list of design decisions, a **Run** section with
  `docker compose up -d --build` and the URL, and where its data lives.
- **M2** Update the README in the same PR when behaviour, ports, env vars or
  URLs change.

### Reviews

These cover every review: a person's, the PR bot's, an agent's self-review
and the nightly sweeps.

- **V1** Review against every rule that applies, to the letter, not only the
  rules a tool checks. A rule's *Check:* sentence names only the part a tool
  enforces; check the rest by reading the code. Something a rule requires
  that's missing (a `LOG_LEVEL` setting, K20) is a break, just like something
  it forbids that's there.
- **V2** A rule break is never a matter of taste, however small: a raw `2px`
  in a CSS Module breaks U4 as much as a raw colour does. It stands unless a
  comment next to it gives the reason.
- **V3** Review assuming there are bugs. Follow each entry point (route, job,
  loop, UI action) through empty, huge, malformed and duplicate input, two
  requests at once, an outside service that's slow or down, old or
  unexpected rows, time zones and daylight saving, and every error path.
- **V4** Report every real problem, small ones included, each with its
  evidence (`file:line`, the code quoted) and how sure you are. Don't leave
  out a problem because you aren't certain; leave out only guesses with no
  evidence.

### Pull requests

- **P1** One change per PR. Say what changed, why, and how it was checked
  (ran it, curl'd `/healthz`, opened it on a phone...).
- **P2** If deploying needs anything beyond `docker compose up -d --build`
  (a new `.env` value, a data migration, a Caddy reload), list the steps.
- **P3** Don't stack PRs the owner merges: those branch from `main` and
  target `main`. If a change needs another open PR, wait for that one to
  merge and branch from `main` again, or make them one PR. Stacking is fine
  when an agent (a wfa PM) merges the whole stack: each PR targets the
  branch below it, the body states the merge order (`#45 → #42 → #44`),
  and the agent merges bottom-up, retargeting the next PR to `main` as each
  one lands.

### Changing these standards

- **E1** When the owner corrects how something should be done, or says
  "always" or "never" do something, and it applies beyond this one change:
  apply it now, then propose it as a rule here with the `standard` skill.
  Corrections that only fit one repo go in that repo's own `CLAUDE.md` and
  `AGENTS.md`, outside the synced block.
- **E2** If a rule can be enforced by a lint rule, test or hook, the PR that
  adds the rule says how, and adds the check once the tooling exists.
- **E3** Wherever a person reads a rule ID (a chat reply, a PR body or
  comment, a review, a finding, a report), give a few words saying what the
  rule is next to it: `N4 (no service exposed to the internet)`, never a
  bare `N4`. No script can tell a bare ID from a described one, so
  reviewers check this one.
<!-- END turple-dog/engineering-standards -->
