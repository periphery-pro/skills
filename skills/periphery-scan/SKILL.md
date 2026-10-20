---
name: periphery-scan
description: Runs a Periphery scan for unused Swift code in a project where Periphery is already set up. Use when a user asks to run Periphery, scan for unused or dead Swift code, or find and remove unused code with Periphery. If Periphery isn't installed, signed in or configured for the project, use periphery-scan-setup instead.
---

# Periphery Scan

## Overview

Run `periphery scan` with the `agent-prompt` output format, which lists each result with removal guidance written for agents. The project's `.periphery.yml` supplies the options, so the scan itself needs no decisions. If anything about the setup is missing, hand over to the periphery-scan-setup skill rather than configuring Periphery here.

## Steps

### 1. Check that Periphery is set up

**Bazel** (`MODULE.bazel`, `WORKSPACE` or `BUILD.bazel`): look in the BUILD files for a runnable `rules_periphery` target, usually `scan_auto` (often `//:scan_auto`) or `scan`. If there's none, go to periphery-scan-setup.

**Everything else:**
- Run `periphery auth status`.
  - If `periphery` isn't found, or the `auth` command doesn't exist (an outdated or open-source release), go to periphery-scan-setup.
  - If it exits non-zero because there's no credential, the user must sign in. `periphery auth login` opens a browser, so ask the user to run it themselves (in Claude Code: `! periphery auth login`). Never ask for an access token's value.
- Look for `.periphery.yml`, or `.periphery.yaml`, in the project root.
  - If there's neither, a Swift package (`Package.swift` and no Xcode project) scans fine with the defaults.
  - Anything else without a config file isn't configured: go to periphery-scan-setup.

To go to periphery-scan-setup, tell the user what's missing and use that skill. If it isn't installed, they can add it with `npx skills add periphery-pro/skills --skill periphery-scan-setup`, or set up Periphery by following https://periphery.pro/docs/running-a-scan.

### 2. Scan

```sh
periphery scan --format agent-prompt > <output file>
```

For Bazel: `bazel run <scan target> -- --format agent-prompt > <output file>`.

- Save the output to a file outside the project, such as in a temporary directory, so you can read it without scanning again.
- Scanning builds the project, which can take many minutes. Tell the user, and run it with a long timeout or in the background.
- Don't add other options unless the user asked for them. The config file holds the project's choices, and options passed on the command line override them.

### 3. Report, then remove if asked

If the output says `No unused code detected.`, tell the user and stop.

Otherwise, the output is a removal task: results, a `## Summary` section and removal guidance. Give the user a brief summary first:
- the number of results, broken down by kind. Each numbered result starts with its kind in brackets, and indented lines under it are details, not separate results. Count them, rather than tallying by eye: `grep -oE '^[0-9]+\. \[[^]]+\]' <output file> | sed 's/^[0-9]*\. //' | sort | uniq -c`
- the removable unused code estimate from `## Summary`, quoted. A trial gets two lines, for the trial results and the full results; quote both.
- if there's a "Trial Mode" notice: what share of the files the results cover, and how many issues are hidden

Then:
- If the user asked only to scan or find unused code, offer to remove the results. Don't start until they agree.
- If they asked to remove unused code, carry on.

Before removing, suggest a clean working tree or a new branch, since removal touches many files. Then follow the removal task and guidance in the output. If removing code leaves a file empty, delete the file and remove its references from the Xcode project, or the build fails. When you're done, tell the user what you removed and offer to scan again: removing code can leave other code unused.

### 4. If the scan fails

Tell the user what failed, quoting the key error line.
- `No account credential found`: the user must sign in, as in step 1.
- Errors about the project, schemes or targets, such as a project or scheme that can't be found, mean the configuration is out of date. Go to periphery-scan-setup.
- Otherwise, read https://periphery.pro/docs/troubleshooting for known issues. Propose any change to the project, build settings or `.periphery.yml`, and wait for the user to confirm it.

If you can't resolve it, offer to open an issue at https://github.com/periphery-pro/issues, following periphery-scan-setup's failure step. The tracker is public, so the issue must be anonymous: replace every project, scheme, target, module, file, type, user, path and company name with placeholders, include no source code, and post only after the user approves the full text.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Writing or editing `.periphery.yml` to get a scan working | Hand over to periphery-scan-setup |
| Adding `--schemes`, `--retain-public` or other options on the command line | Let the config file supply them |
| Removing code when the user only asked for a scan | Summarize, then offer removal |
| Revalidating each result before removing it | The output says to treat results as authoritative; follow it |
