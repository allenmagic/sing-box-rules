# Rule Sync Local JSON Compilation Design

## Summary

Extend `.github/workflows/rule-sync.yml` so the workflow compiles every `config/*.json` file into a same-name `.srs` file with `sing-box`, stages those generated rule sets together with the downloaded upstream `.srs` files, and pushes only `.srs` artifacts to the target Gitea repository under `rules/`.

Compilation failures for individual JSON files must not fail the workflow. The workflow should emit warnings for failed files, continue compiling the rest, and still push any successfully generated `.srs` files plus the downloaded upstream `.srs` files.

## Current State

- The workflow downloads three upstream `.srs` files into a temporary `rules/` directory.
- The workflow clones or creates the target Gitea repository and copies `rules/*.srs` into `gitea-repo/rules/`.
- The repository contains local source rule definitions in `config/`, currently including `config/ai-proxy-list.json`.
- README content is partially out of sync with the actual workflow: the cron explanation and destination path text do not fully match the current YAML.

## Goals

- Compile every `config/*.json` file to `rules/<basename>.srs` during the workflow run.
- Keep the final push contract unchanged: only `.srs` files are pushed to `gitea-repo/rules/`.
- Continue processing other JSON files when one compile fails.
- Surface compile failures clearly in workflow logs as warnings.
- Update documentation so operators understand both upstream downloads and local JSON compilation.

## Non-Goals

- Pushing source JSON files to Gitea.
- Introducing a separate build script or local test harness.
- Changing the existing Gitea repository bootstrap logic.
- Changing the target sync directory from `rules/`.

## Proposed Workflow Changes

### 1. Install `sing-box`

Add a dedicated step after the upstream download step to install a pinned `sing-box` Linux binary inside the GitHub Actions runner.

Design constraints:

- Use an explicit pinned version rather than `latest` to keep runs reproducible.
- Download from the official GitHub release artifacts for Linux AMD64.
- Place the executable in a path available to subsequent workflow steps.

### 2. Compile Local JSON Rule Sets

Add a step after `sing-box` installation that:

- Checks whether `config/` exists.
- Iterates over `config/*.json`.
- For each JSON file, computes the target output as `rules/<basename>.srs`.
- Runs `sing-box rule-set compile --output "<target>" "<source>"`.

Behavior rules:

- If no JSON files exist, log a short message and continue successfully.
- If compilation succeeds, retain the generated `.srs` in `rules/`.
- If compilation fails, do not exit the step immediately.
- Emit a GitHub Actions warning for the failed file and continue to the next file.
- At the end of the step, emit a summary warning if one or more files failed.

### 3. Preserve Existing Push Model

Keep the Gitea sync step structurally the same:

- Continue cloning or creating the target repository.
- Continue copying `rules/*.srs` into `gitea-repo/rules/`.
- Continue committing and pushing only when the target repo contents actually changed.

This preserves compatibility with the current destination layout and avoids mixing source files into the output repository.

## Failure Handling

### Download Failures

Downloaded upstream `.srs` files should continue using the existing failure behavior. If an upstream download command fails, the workflow run fails as it does today.

### Local Compilation Failures

Compilation failures are soft failures:

- The failed JSON file does not produce or update an `.srs` artifact.
- The workflow logs a warning for that file.
- Other JSON files continue compiling.
- The workflow still proceeds to the push step.

This ensures that one broken local source file does not block the delivery of other valid local rules or downloaded upstream rules.

## Logging

Workflow logs should make these states obvious:

- `sing-box` installation version.
- Which JSON file is currently being compiled.
- Which files compiled successfully and their target names.
- Which files failed and were skipped.
- Whether the workflow found no local JSON files.

GitHub Actions warning annotations should be used for compile failures so they are visible in the run summary.

## Documentation Updates

Update `README.md` to reflect the actual workflow behavior:

- Mention that the workflow now handles both downloaded `.srs` files and locally compiled `config/*.json` rule sets.
- State clearly that only `.srs` artifacts are pushed to Gitea.
- Document that local JSON compile failures produce warnings and do not stop the sync.
- Correct the destination path description to `rules/`.
- Correct the cron explanation so it matches the actual cron expression in `.github/workflows/rule-sync.yml`.

## Validation Plan

Because this repository has no automated test suite, validation is workflow-focused:

1. Review the rendered workflow YAML for syntax regressions.
2. Review the shell loop to confirm it correctly handles zero, one, and many JSON files.
3. Confirm the compile command maps `config/foo.json` to `rules/foo.srs`.
4. Confirm failed compiles produce warnings without stopping the job.
5. Confirm the push step still copies only `rules/*.srs`.
6. Manually trigger the GitHub Actions workflow after merge to verify real runner behavior and the `sing-box` download path.

## Implementation Notes

- Keep shell variable expansion quoted as `"${VAR}"`.
- Prefer simple POSIX-compatible shell patterns in the workflow step.
- Avoid printing authenticated remote URLs or tokens in logs.
- Keep the workflow changes localized to `.github/workflows/rule-sync.yml` and the matching README updates.
