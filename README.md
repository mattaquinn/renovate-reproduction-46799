# 46799

Minimal reproduction for [Renovate Discussion #46799](https://github.com/renovatebot/renovate/discussions/46799).

## Current behavior

In a repository that uses Central Package Management (`Directory.Packages.props`), every project file is a dependent of the props file. When Renovate updates `Directory.Packages.props`, the `nuget` manager's `updateArtifacts` ([`lib/modules/manager/nuget/artifacts.ts`](https://github.com/renovatebot/renovate/blob/main/lib/modules/manager/nuget/artifacts.ts)):

1. builds one candidate `packages.lock.json` path per project file, then
2. passes the whole list to `getFiles()` ([`lib/util/git/index.ts`](https://github.com/renovatebot/renovate/blob/main/lib/util/git/index.ts)), which is a serial loop that runs one `git show origin/<branch>:<path>` per path.

This repository has no `packages.lock.json` files, so every one of those `git show` calls fails, and only then does `updateArtifacts` log `No lock file found for package or dependents` and return `null`.

The number of git invocations is therefore proportional to the number of project files, once per branch. Process spawn cost is much higher on Windows (about 90 ms each in our production repository) than on Linux, but the invocation count is the same on every platform.

## Expected behavior

Lock file discovery should cost a constant number of git invocations, not one per project file.
When none of the candidate `packages.lock.json` paths appear in `scm.getFileList()`, Renovate should not run any `git show` for them.

## Link to the Renovate Discussion

<https://github.com/renovatebot/renovate/discussions/46799>

## Repository contents

| Path                                    | Purpose                                                                                                                                            |
| :-------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Directory.Packages.props`              | One central package version, intentionally outdated (`Newtonsoft.Json` 12.0.1) so Renovate proposes two branches: `…-12.x` (patch) and `…-13.x` (major). |
| `src/Project001` … `src/Project100`     | 100 identical stub projects. Each has a versionless `PackageReference`. The number of project files is the variable under test.                    |
| `renovate.json`                         | Minimal config: only the `nuget` manager is enabled.                                                                                               |
| _(no `packages.lock.json` anywhere)_    | The condition that makes every probe fail.                                                                                                         |

## Reproduce

Run Renovate with debug logging against this repository. `--dry-run=full` still runs `updateArtifacts` but pushes nothing. Any token that can read this public repository works.

```bash
export LOG_LEVEL=debug
export RENOVATE_TOKEN=<token with read access>
npx renovate@44.149.0 --platform=github --dry-run=full <owner>/<this-repository> 2>&1 | tee renovate.log
grep -A 12 'Git operations statistics' renovate.log
```

Look at the `other` bucket. `git show` is not tracked separately, so it is counted there.

Expect `other.count` to be about 100 per branch (about 200 for the two branches here) higher than a repository with the same dependencies but no project files.

To confirm the scaling, delete `src/Project051` … `src/Project100`, commit, and run it again. `other.count` drops by about 100.
