# AGS5 Control work log

## Approved scope
- Android Expo app; black/gold; multi-user.
- Each user runs their own Windows MT5 + AGS5 EA.
- Secure pairing; Start/Pause; authoritative telemetry; no MT5 passwords.
- Windows install / MetaEditor compilation are user-side verification.

## Plan
- [x] Better Auth managed Google + email/password and DB schema.
- [x] Owner-scoped accounts, pairing, command and heartbeat API.
- [x] Native mobile dashboard, trades, accounts, setup guide, authentication.
- [x] Updated EA: HTTPS pairing + heartbeat, fail-closed entry gate.
- [x] Build, typecheck, API isolation tests, mobile browser verification.
- [x] Deliver mobile app and EA source; clearly state remaining MT5 test gap.

## Safety decisions
- Remote pause does not liquidate or disable trailing.
- Start rejected when terminal stale/Algo Trading off; acknowledgement required.
- Backend connection lost: lease expires and new entries pause.
- At most two order submissions daily; fixed symbol/lot and magic retained.
- No live accounts or orders used during sandbox testing.

## Verification outcomes
- 26 API checks passed: authentication, owner isolation, token/pair replay denial,
  pending vs acknowledged commands, offline/Algo Trading-off Start denial, revoke.
- Mobile browser: navigation, forms, guide; full real API signup/pair UI/signout passed.
- Generated test users and simulated terminal/audit records cleaned up.
- App build + all TypeScript checks + lint passed; Expo web production export passed.
- EA source static checks passed, NOT MetaEditor compilation or broker backtest.
- Native Android installation and Google managed sign-in not device-tested.
- Template TS7/Expo alias resolver mismatch fixed using relative mobile imports.
- Runtime services: web/API 4200 and mobile 4300, kept in tmux.

## Expo SDK 58 upgrade
- Installed Expo 58.0.20, React Native 0.88.0, React 19.3.0 and matching Expo modules.
- Matched web React version; Bun hoisted linker removes duplicate native modules.
- Migrated obsolete splash config to splash-screen plugin; removed obsolete
  edgeToEdgeEnabled/newArchEnabled fields; corrected iOS bundle ID syntax.
- Preserved Android package, managed scheme/extra, provider chain, EA/backend behavior.
- Explicit SPA web output retained; tab icon colors adapted to RN strict types.
- Added Babel 7 parameter lowering for oRPC destructured rest syntax unsupported
  by Hermes; Android export now produces Hermes bytecode successfully.
- Expo Doctor 20/20; expo install --check; lint, all typechecks and root build pass.
- SDK58 web export and Android Hermes bundle export pass.
- Mobile browser navigation/forms/guide and actual API signup/account/pair/signout pass.
- 26 API regression checks pass; generated API/UI fixtures cleaned up.
- Manifest reports sdkVersion 58.0.0. Physical Android/Expo Go and native Google
  login remain untested; Expo Go must support SDK58.
- Local SDK54 configuration/lockfile backup: /home/user/ags5-sdk54-backup.
- SDK58 CLI reads original process env for packager proxy, so mobile tmux startup
  explicitly sources the existing mobile proxy env before starting the server.

## Approved M1 + M5 change — EA v2.10
- User chose independent M1/M5 closed-bar signals; M5 wins simultaneous signals.
- Separate indicator sets and fresh-candle tracking; only one entry attempt per tick.
- Each entry uses its own timeframe ATR for spread filter and initial SL/TP.
- Existing M5 ATR trailing retained for both entry types, including after restart.
- Shared 0.01 lot, two daily request slots, one gold position, lock and bridge guards retained.
- Included diagnostics for both signal timeframes, skipped-entry reasons and broker results.
- Reset cached device identifiers on initialization before loading current account credentials.
- Original v2.00 source preserved in mt5/AGS5_App_v2_00_backup.mq5.
- Main EA source and public downloadable source are identical v2.10 files.
- Mobile strategy/guide text updated; stale request notice hidden after pending ends.
- Heartbeat JSON/schema rejection now distinct from internal DB/network failure (503).
- Live server logs confirmed internal processing failures. Root cause and sustained
  recovery are NOT verified; the backend change diagnoses, not guarantees a fix.
- Verified: 13 source-invariant/Python model tests; 31 API regression checks;
  3 isolated failure-classification tests; lint/typecheck/build; Android Hermes
  export; mobile browser navigation/forms/guide; public EA download byte match.
- No Windows MetaEditor compile, MT5 Strategy Tester run or broker fill verified.
- User must remove all older/diagnostic instances and demo-test v2.10 before live use.

## Approved Bollinger/Stochastic replacement — EA v2.20
- Implement BB(20,2), Stochastic(14,3,3 SMA low/high); remove EMA/RSI entry logic.
- Touch current/prior closed candle; latest close strictly inside both bands.
- Previous K/D both <=20 for bullish cross or >=80 for bearish cross; current cross strict.
- M1 additionally requires matching closed M5 middle-band slope (shifts 1/2).
- M5 independent, simultaneous M5 priority, shared guards and M5 ATR trailing retained.
- v2.10 backup preserved; public download and mobile copy updated.
- Verified: 18 source-invariant/Python signal-model tests; 31 API checks with simulated
  terminals; 3 isolated backend failure tests; root lint/typecheck/build; Android
  Hermes export; mobile browser navigation/forms and new strategy/bridge FAQs.
- Public HTTPS EA download byte-identical to main v2.20 source.
- Direct v2.10 comparison confirms unchanged order gates, bridge, trailing and
  fresh-candle/M5-priority routing (apart from chart display wording).
- Reviewed mobile guide screenshot; no browser runtime errors.
- No MetaEditor compilation, MT5 Strategy Tester run, broker fills or physical
  Android install verified. Python model/source tests are not MQL execution.
- Bridge root cause/sustained recovery still unverified; no live trades placed.

## Bridge connection-reset fix — 2026-10-28
- Server logs now confirm QUERY_FAILED / ECONNRESET / SOCKET (not bad telemetry).
- Matches idle HTTP socket reuse at approximately five-second Turso/heartbeat cadence.
  Supporting upstream report: https://github.com/oven-sh/bun/issues/45138
- Added app-owned database connection/transport, leaving template __client and
  required database/index export untouched. Auth, terminal routes, webhooks and
  audit service now use the application connection.
- Database HTTP requests use fresh connections (Bun keepalive:false + Connection:
  close for Node), preserve cross-fetch Request bodies/headers, retain cancellation,
  and have 1500ms per-upstream-request deadlines. No database POST/write retries.
- Initial native-fetch adapter failed with ERR_INVALID_URL because Hrana's Node
  Request is from cross-fetch; fixed by normalizing URL/method/headers/body by shape.
- Verified: root lint/typecheck/build; five isolated transport tests; three isolated
  heartbeat-failure tests; 31 local API checks; 31 public HTTPS API checks including
  18 consecutive simulated heartbeats spaced 4985ms apart. Public heartbeat latency
  min 667ms / mean 790ms / max 1737ms. Fixture users/terminals/audits cleaned up.
- Mobile navigation/forms/new strategy copy and actual signup/pair/signout browser
  checks passed; 18 strategy/source checks passed. EA remains unchanged v2.20.
- Connected real terminal showed a fresh heartbeat within the last 20 seconds
  after synthetic fixtures were removed. This is current recovery evidence, not
  a long-duration availability guarantee or proof of broker execution.
- No remote Start/Pause commands sent to real terminals; no broker trades placed
  by these tests. The 20-second lease, replay/revoke/ack checks remain unchanged.
- Publishing is platform-owned and no publish action is available to the agent.
  Permanent publication still requires the user to press Publish in Runable.
