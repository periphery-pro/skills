# Upgrading From the Open-Source Version

Read this when step 2 finds the legacy open-source Periphery: `periphery auth --help` fails, or homebrew-core's `periphery` formula is installed. The rest of the main skill still applies afterwards. This file covers what's different for someone moving from the open-source version.

## 1. Explain the switch, then ask

The user may not know the two versions differ. Before you change anything, tell them:
- The current Periphery replaces the open-source version, and both install a `periphery` command, so the old one has to be removed.
- Scanning needs a Periphery account: `periphery auth login` for local scans, or an access token in CI. Point them to https://periphery.pro for plans.
- Public GitHub repositories can use Periphery for free, but only through the official GitHub workflow (https://periphery.pro/docs/open-source). Free scans don't cover local runs.

Ask whether they want to switch. If they decline, stop setup here; don't leave them with half of each version.

## 2. Replace the binary

Follow step 2 of the main skill: use the published installation docs, including the removal of the legacy version they call for. If the open-source version came from somewhere other than Homebrew, offer to remove it with the tool that installed it, for example `mint uninstall`. Then check `which -a periphery` so no old binary comes first on the `PATH`. Bazel projects get the new binary through `rules_periphery` instead (see section 4), so they only need the old one removed.

## 3. Migrate `.periphery.yml`

Most keys are unchanged. These open-source keys no longer exist. The current CLI only warns about an unknown key and otherwise ignores it, so a stale key silently loses its setting:

| Open-source key | Now |
|---|---|
| `bazel`, `bazel_filter`, `bazel_query`, `bazel_index_store`, `bazel_check_visibility` | Bazel moved to `rules_periphery`; see below |
| `xcode_list_arguments` | No equivalent; remove it |
| `skip_schemes_validation` | No equivalent; remove it |

Show the user the current file and your proposed version, saying what each removed key did, and ask before writing. Anything else you're unsure of, check against `periphery scan --help`. After the first scan, look for `invalid key` warnings.

Comment commands (`// periphery:ignore` and friends) work as before.

## 4. Bazel users

The open-source version scanned Bazel projects with `periphery scan --bazel`. Now Bazel goes through `rules_periphery`, so follow [bazel.md](bazel.md). Carry the old settings over to `scan_auto` attributes:

| Open-source key | `scan_auto` attribute |
|---|---|
| `bazel_filter` | `filter` |
| `bazel_query` | `query` |
| `bazel_index_store` | `global_indexstore` |
| `bazel_check_visibility` | `check_visibility` |

Remove the `bazel*` keys from `.periphery.yml` once they've moved.

## 5. Find other places that use the open-source version

These can reinstall the open-source version or keep calling it. Search the repo for them, for example with `git grep -n -i periphery`:
- `Brewfile` (`brew "periphery"`) and `Mintfile` (`peripheryapp/periphery`)
- `Package.swift` dependencies or plugins from `github.com/peripheryapp/periphery`
- Xcode Run Script build phases in `project.pbxproj`
- CI workflows and scripts that install or run Periphery

List what you find. Offer to update each one, and change nothing until the user agrees. CI also needs authentication now: an access token (https://periphery.pro/docs/continuous-integration) or, for public repositories, the free workflow (https://periphery.pro/docs/open-source).
