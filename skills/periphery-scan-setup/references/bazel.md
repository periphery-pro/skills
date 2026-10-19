# Bazel

Bazel projects use [rules_periphery](https://github.com/periphery-pro/rules_periphery) instead of `periphery scan` directly. Docs: https://periphery.pro/docs/build-systems#bazel.

Steps 4 (options), 5 (`.periphery.yml`), 7 (failures) and 8 (finish) of the main skill still apply. Step 2 is replaced by the module setup and sign-in below, since `rules_periphery` provides the binary. Step 3 (schemes) does not apply: `scan_auto` finds the top-level targets itself.

## Existing setups

If `MODULE.bazel` already uses `rules_periphery`, check what it pins before changing anything:
- A `binary_archive` URL from `github.com/peripheryapp/periphery` is the legacy open-source version. Explain the switch as in section 1 of [upgrading-from-open-source.md](upgrading-from-open-source.md), since it means signing in to an account, then point the archive at `cli-releases` as below.
- Older `rules_periphery` or CLI versions: offer to update them.
- `periphery_bin` missing from `use_repo`: add it, or signing in won't be possible.

Keep the rest of the user's setup, such as `dev_dependency` and existing `scan_auto` attributes. To check the configured binary, run `bazel run @periphery_bin//:periphery -- version` and `-- check-update`.

## 1. Add the module

These are edits to the user's build files. Show the exact changes and wait for confirmation.

In `MODULE.bazel`:

```starlark
bazel_dep(name = "rules_periphery", version = "<rules_version>", dev_dependency = True)

periphery = use_extension("@rules_periphery//:extensions.bzl", "periphery", dev_dependency = True)
periphery.binary_archive(
    url = "https://github.com/periphery-pro/cli-releases/releases/download/<cli_version>/periphery-cli_<cli_version>_<os>_<arch>.zip",
    sha256 = "...",
)
use_repo(periphery, "periphery_bin", "periphery_generated")
```

- `<os>` is `macos` or `linux`, and `<arch>` is `arm64` or `x86_64`, for the machine that runs Bazel.
- `rules_periphery` and the CLI are versioned separately. For `<rules_version>`, use the latest version in the Bazel Central Registry (https://registry.bazel.build/modules/rules_periphery). For `<cli_version>`, use the latest CLI release: `gh release list --repo periphery-pro/cli-releases --exclude-drafts --limit 1`, or `curl -s "https://api.github.com/repos/periphery-pro/cli-releases/releases?per_page=1"` (the `tag_name` field). Don't use the `/releases/latest` endpoint, which skips pre-releases. If `MODULE.bazel` already pins older versions, offer to update them.
- Get the `sha256` by downloading the archive and running `shasum -a 256` on it. A placeholder or guessed checksum makes the Bazel fetch fail.
- Alternatively, if Periphery is already installed, offer `periphery.local_binary(path = "...")`. It's quicker to set up but isn't reproducible on other machines.
- `periphery_bin` in `use_repo` makes the binary runnable for signing in.

## 2. Sign in

Check first with `bazel run @periphery_bin//:periphery -- auth status`, which exits non-zero when there's no credential. If they aren't signed in, ask the user to run this themselves, since it opens a browser to sign in to their account (in Claude Code, prefix it with `!`):

```sh
bazel run @periphery_bin//:periphery -- auth login
```

Scans run with `bazel run` use the stored credential. On CI, use a `PERIPHERY_TOKEN` access token instead (https://periphery.pro/docs/authentication); never ask for its value or write it into a file.

## 3. Add the scan target

In the root `BUILD.bazel`:

```starlark
load("@rules_periphery//:defs.bzl", "scan_auto")

scan_auto(
    name = "scan_auto",
)
```

`scan_auto` scans all top-level targets. If the user wants to limit it, use `filter` (for example `filter = "^//App"`), or `query` to replace the query entirely, such as to leave out `manual` targets.

## 4. Write `.periphery.yml`

Do this as in the main skill, but without `project` or `schemes`. Periphery runs from the workspace root under `bazel run`, so it finds `.periphery.yml` there.

## 5. Run the first scan

Bazel creates `bazel-*` symlinks and a `MODULE.bazel.lock` in the workspace. If the repo doesn't ignore the symlinks yet, suggest adding `bazel-*` to `.gitignore`.

```sh
bazel run //:scan_auto -- --format agent-prompt > <output file>
```

Bazel-specific failures:
- Visibility errors: by default the target runs with `--check_visibility=false`. Leave that default for the first scan.
- A target that fails to build when built directly (for example because of a transition): narrow the `query` to leave it out.
- Run with `--verbose` (`bazel run //:scan_auto -- --verbose`) to see the generated query.
