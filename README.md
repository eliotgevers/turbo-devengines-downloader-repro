# Automatic local Turbo download rejects strict Bun metadata

Upstream report: [vercel/turborepo#14358](https://github.com/vercel/turborepo/issues/14358).

This workspace has no application dependencies. Keep it without node_modules to exercise the bootstrap path.

Prerequisites: Bun 1.4.2, Node, npm/npx, and a separately installed global/native Turbo binary. npm 10.9.8, 11.19.0 and 12.2.0 all reproduced the failure. Do not install Turbo into this workspace before the failing check: a local install bypasses the downloader.

The checked-in lockfile pins target Turbo 2.11.5. The bootstrap binaries tested were Turbo 2.11.7 and the currently tagged canary 2.11.5-canary.5. The error happens in their downloader before target Turbo starts.

Install a launcher outside the workspace, or use an existing external binary:

```sh
bootstrap="$(mktemp -d)"
printf '%s\n' '{"private":true}' > "$bootstrap/package.json"
bun add --cwd "$bootstrap" --exact turbo@2.11.5-canary.5
export TURBO_BIN="$bootstrap/node_modules/.bin/turbo"
```

From this workspace, without running a normal installation:

```sh
TURBO_DOWNLOAD_LOCAL_ENABLED=1 "$TURBO_BIN" run build --dry=json
```

Expected: the declared Bun workspace remains usable and the locked Turbo version runs.

Actual: Turbo launches `npx -y turbo@2.11.5 --skip-infer run build --dry=json --` in the repository root. npm rejects the Bun `devEngines.packageManager` declaration with EBADDEVENGINES.

Controls:

```sh
# Avoid downloading; runs the outer global version instead of the locked target.
TURBO_DOWNLOAD_LOCAL_ENABLED=0 "$TURBO_BIN" run build --dry=json

# Install root + selected app, preserving strict metadata and the download setting.
cd apps/app
bun install --frozen-lockfile --filter repro --filter .
TURBO_DOWNLOAD_LOCAL_ENABLED=1 "$TURBO_BIN" run build --force
```

An app-only `bun install --frozen-lockfile --filter .` does not install root Turbo. Adding `onFail: "warn"` permits the npm helper, but relaxes the policy rather than correcting the subprocess selection.

Both launcher versions fail before the locked target starts. The root-plus-app installation lets the external launcher delegate to the locked local Turbo and run the tiny task. No framework, deployment service or credentials are required.
