# Android startup repair — JayFx AGS

## What was found

The public `jasonjjordan977-art/jayfx-app` repository at commit
`5940af477462ef386f3939be2b25038f0500f985` contained only nine configuration files.
There was no `app/` route directory, `components/`, `lib/`, assets, backend workspace
or dependency lockfile. Its `app.json` omitted the backend URL, managed-auth IDs,
native scheme and Expo plugins. Its Metro configuration assumed a monorepo two
directories above a repository that was actually flat.

These are confirmed source problems. Without an Android crash log or a build
commit identifier, they do not prove the exact native exception in the linked APK.

## What this replacement contains

- Full original AGS5 Control monorepo, including all screens and assets.
- Latest BB/Stochastic EA v2.20 and backend connection-reset fix.
- Restored Expo Router, native modules/plugins, auth configuration and API URL.
- Your Expo owner `jayfx55s-team`, project slug `ags`, project ID
  `3e86484a-57d8-465e-b34a-9ac545c0906c`, and Android package `com.jayfx55s.ags`.
- App display name `Aggressive Gold Sniper`, version 1.0.1, local Android version
  code 2; the preview profile enables automatic incrementing for subsequent builds.
- Your internal-distribution APK profile and production profile.
- The existing native auth scheme is retained because the backend trusts it.

Do not flatten `packages/mobile` into the repository root. It belongs at that path
inside the monorepo and imports shared API types from `packages/web`.

## Replace the incomplete GitHub source

1. Back up your edited repository first.
2. Extract this ZIP. Put its **contents**, not an extra enclosing folder, at the
   root of your local GitHub checkout. Retain your checkout's `.git` directory.
3. Remove the obsolete root mobile-only `app.json`, `eas.json`, `metro.config.js`,
   `babel.config.js`, and `__env.d.ts` from that checkout. The mobile versions now
   live inside `packages/mobile`. Use the supplied root `package.json` and
   `tsconfig.json`, not your prior mobile-only versions. Do not delete the new
   `packages/mobile` versions.
4. Do not commit `.env`, credentials or signing files. The previous repository's
   Expo packager proxy `.env` is sandbox-only and is not required for the APK.
5. From the monorepo root, run `bun install --frozen-lockfile`, then
   `bun run typecheck` and `bun run build`.
6. Commit/upload the **whole monorepo**, including `bun.lock`, `.runable/ports.json`,
   assets and all `packages` directories, excluding dependencies and secrets.

## Rebuild on your own machine / EAS cloud

From `packages/mobile` in the replacement repository:

```bash
bunx eas-cli login
bunx eas-cli build --platform android --profile preview
```

Use your existing Expo account and signing credentials for `com.jayfx55s.ags`.
Do not regenerate signing keys casually; an update must use the same signing key.
This is an EAS cloud build, not a build performed in the Runable sandbox. If using
Runable's managed mobile project, its Publish control is the supported build path.

Install the **new** completed APK. If it cannot replace the old one, check signing
identity and version codes before uninstalling; uninstalling clears local sessions.
An old APK will not gain these source changes merely because GitHub was updated.

## Backend and auth

The mobile API URL currently remains the original public Runable preview backend.
This allows reuse of the existing app's backend configuration, but it is not
permanent independent hosting. For your own deployment, follow
`BUILD-AND-PUBLISH.md`: host the backend/database, update API/auth URLs and scheme
registrations, then rebuild. No private backend credentials are in this ZIP.

Google sign-in still depends on Runable Managed Auth. Email/password also requires
the backend. No account migration or independent Google OAuth setup was performed.
No live trading settings or real terminal commands were changed by this repair.

## If the newly rebuilt APK still closes

Connect the phone to a computer with Android platform-tools and USB debugging.
Capture logs around one launch; these commands are for your computer:

```bash
adb logcat -c
adb logcat -v threadtime > ags-startup.txt
```

While the second command runs, launch AGS, wait for it to close, then stop the
capture with Ctrl+C. Share the `FATAL EXCEPTION`, `AndroidRuntime`, `ReactNativeJS`
or Hermes error section, with tokens/account information redacted. A device log
is needed to diagnose a remaining native crash accurately.

## Verification

- Fresh `bun install --frozen-lockfile` passed.
- Root lint, all workspace typechecks and root build passed.
- Android export passed: 1,599 modules, 81 assets and a 4.2MB Hermes bundle.
- Source/config checks confirm routes, assets, backend URL, auth scheme and your
  retained Expo project ID and Android package.

Source checks and Hermes export are not the same as building/installing the APK
or verifying Honor hardware.
No replacement APK, native signing, store publishing or physical-device launch
has been performed here.