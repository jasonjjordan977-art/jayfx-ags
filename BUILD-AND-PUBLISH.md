# AGS5 Control — complete source export

This archive contains the Expo Android/iOS client, Bun/Hono backend, React web
scaffold, Electron desktop scaffold, MT5 Expert Advisor v2.20, assets, schemas,
tests, lockfile and template tooling. It is source code, **not an APK/AAB/IPA**.

The developed controller UI is in `packages/mobile`; the React web and Electron
packages are scaffolds, not equivalent copies of the AGS5 mobile interface.

## Important: standalone does not mean offline

The installed phone app is a remote controller. Windows MT5 executes the EA.
Pairing, login, Start/Pause and telemetry require a publicly reachable HTTPS
backend and a Turso-compatible database. Keep the backend and Windows MT5 online.
No user database, credentials, device tokens or account data are included.

Google sign-in currently uses Runable Managed Auth, which remains a dependency
in both client and server. This ZIP does **not** migrate it to your own Google
OAuth project. Email/password also uses the hosted backend. For fully independent
hosting, review/replace the managed-auth integration in `packages/web/src/api/auth.ts`
and `packages/mobile/lib/auth.ts`, and configure your own OAuth credentials,
redirects and native deep links before enabling Google sign-in.

## 1. Install dependencies

Use the repository as a monorepo; do not copy just `packages/mobile` out of it.
It imports the backend's types through the `@template/web` workspace.

From the extracted `ags5-control` root:

```bash
bun install --frozen-lockfile
```

The verified sandbox runtime was Bun 1.4.4; the inherited `packageManager` field
still says Bun 1.3.14. Expo 58.0.20, React Native 0.88.0 and React 19.3.0 are
locked in this export. Do not upgrade dependencies blindly. Preserve the hoisted
Bun linker and Babel parameter transform used for Hermes compatibility.

## 2. Configure your backend

Copy `.env.template` to a **root** `.env` file and supply your own values.

- `DATABASE_URL`, `DATABASE_AUTH_TOKEN`: your database, not the original app's.
- `BETTER_AUTH_SECRET`: generate a strong private random secret (32+ bytes).
- `WEBSITE_URL`: the backend's final public HTTPS origin, without a trailing slash.
- `APPLICATION_ID`, `VITE_RUNABLE_AUTH_ISSUER`: required while retaining managed
  Runable authentication; use values registered for your application.
- Other template gateway/payment/storage variables are optional unless you use
  those modules. Do not expose API keys through `VITE_` or `EXPO_PUBLIC_` variables.

This export deliberately excludes every actual `.env`, logs, cache, installed
dependencies, generated bundles, signing material and local databases.

For a **new, empty** database, apply the schema from `packages/web`:

```bash
bun run db:push
```

Inspect migration changes before applying them to an existing production database.
The archive does not transfer existing users or paired terminals; a fresh database
requires fresh accounts/pairings.

## 3. Set production URLs and app identity

Before building a phone binary, edit `packages/mobile/app.json`:

- `expo.extra.apiUrl`: your permanent HTTPS backend URL. Its existing value points
  to a Runable preview and is **not** a permanent standalone deployment.
- `expo.extra.applicationId` / `runableAuthIssuer`: keep consistent with the server
  if retaining managed auth.
- `expo.android.package`, `expo.ios.bundleIdentifier`, `expo.scheme`: register
  identifiers you own. Existing identifiers are inherited from this project.
- Configure EAS ownership/project ID and signing under your own Expo account.

`extra.apiUrl` takes precedence over `EXPO_PUBLIC_API_URL` in the current code;
setting that environment variable alone will not replace the existing JSON URL.

Review `trustedOrigins` in `packages/web/src/api/auth.ts`. It currently derives
the preview mobile origin by replacing `-4200.` with `-4300.` and allows the
development localhost origins. For an independently hosted mobile-web site,
explicitly add its exact HTTPS origin and the chosen native scheme. Do not use
wildcard trusted auth origins. Review CORS and auth rate limiting for production.

## 4. Run locally and deploy the backend

From the repository root, in separate shells:

```bash
bun run dev
bun run dev:mobile
```

The fixed development ports are 4200 (API/web) and 4300 (Expo). `__ports.cjs`
requires `.runable/ports.json`, which is included. Keep template-managed `__`
files intact. Native phones need a reachable API, not the phone's `localhost`.

Validate and build:

```bash
bun run lint
bun run typecheck
bun run build
```

Production backend entry point, from the repository root:

```bash
PORT=4200 bun --env-file=.env packages/web/src/server.ts
```

Alternatively, `bun run start` uses the included PM2 configuration. Put the
backend behind an HTTPS reverse proxy or your host's managed TLS, supply secrets
through the host, and configure restart/health checking. `/api/health` checks the
server process, **not** database availability. Database operations must be tested
separately. The backend serves API routes and static web build output; running
the backend alone does not serve the Expo controller UI.

For a browser version of the controller, export the Expo web client from
`packages/mobile` with `bunx expo export --platform web`, host its output separately,
and configure its exact auth origin as described above.

## 5. Build and publish the installed app

Within Runable, use the mobile preview's **Publish** control and connect your Expo
account there. Native app builds are not performed in this sandbox.

For an external source checkout, Expo's documented EAS flow can be run on your
own machine/cloud account. Run EAS commands from `packages/mobile`, not the monorepo
root. The included `eas.json` has development, preview and production profiles.
Configure the project before building; production Android builds normally target
an AAB for Google Play. An APK for direct installation needs an explicit APK build
profile. Signing, store accounts, privacy disclosures and store review are yours
to configure. No signing keys or store credentials are supplied.

Official references:
- https://docs.expo.dev/build/setup/
- https://docs.expo.dev/build-reference/build-with-monorepos/
- https://docs.expo.dev/build-reference/apk/
- https://docs.expo.dev/submit/introduction/

Do not build a binary with the preview API URL if your goal is independent hosting.
Changing the URL in an already installed binary requires a supported update or rebuild.

## 6. Install MT5 EA

Use `mt5/AGS5_App.mq5` (v2.20). Backup EA versions are included for history only;
never attach them alongside the current EA. The public download copy is in
`packages/web/public/downloads/AGS5_App.mq5` and is byte-identical.

1. Remove all older AGS5/diagnostic instances from every chart.
2. Copy the current EA to MT5's `MQL5/Experts`, then compile with F7 in MetaEditor.
3. Attach **one** instance to exact `XAUUSD` on an M5 chart. It scans M1 and M5.
4. Set `InpControlURL` to the deployed backend URL; add it to MT5's WebRequest
   allowlist. Pair with a private one-time code from your newly configured app.
5. Enable Algo Trading, confirm stable heartbeats and wait for Start acknowledgement.

Existing terminal credentials are tied to the old URL/database. Changing those
requires re-pairing; protect/delete only that terminal's appropriate local key file
as needed, and never share keys or pairing codes. Do not delete daily reservation
globals or change magic 505005 to evade the daily limit.

## Strategy and bridge safety

- BB(20,2) + Stochastic(14,3,3), closed candles. M1 needs matching closed M5
  middle-band slope. M5 reversal signals remain independent and have priority.
- ATR stops/spread checks; M5 ATR trailing. Fixed 0.01 lots, one gold position,
  two shared entry requests per broker day. Rejected/uncertain requests consume a slot.
- Pause or bridge failure blocks new entries; it does not close positions or
  disable local trailing. Broker SL/TP remain; trailing requires MT5 running.
- The database connection-reset patch is in `database/connection.ts` and
  `database/transport.ts`. It avoids reused HTTP sockets, supports cross-fetch
  Request normalization and bounded deadlines, and never retries ambiguous writes.

## Verification and remaining gaps

Verified before export: root lint/typecheck/build, Android Hermes bundle export,
18 EA source/model checks, 31 API regression checks, five transport tests, three
failure-classification tests, mobile browser signup/pair/sign-out and navigation.
The patched public bridge passed 18 simulated heartbeats approximately five seconds
apart. These are not a long-duration uptime guarantee.

Not verified: an APK/AAB/IPA build, physical Android install, native Google sign-in,
MetaEditor compilation, MT5 Strategy Tester/broker execution, external hosting or
store publication. Demo/backtest before live use. No profitability promise.

Tests are included, but several Python/browser checks still use the original
`/home/user/ags5-control` path and localhost ports; adapt test-only paths for your
checkout. API tests create and clean up synthetic accounts; use a test database.
One-time old migration scripts in `tests` are historical and should not be rerun
against the current source.

`SOURCE_MANIFEST.json` records SHA-256 checksums for all other archived files.