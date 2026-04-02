# Repository Guidelines

## Project Structure & Module Organization

- `.github/workflows/rule-sync.yml`: the only automation entrypoint. It downloads upstream `.srs` files, clones or creates the target Gitea repository, and syncs them into `rules/`.
- `README.md`: operator-facing setup and workflow summary.
- `AGENTS.md`: contributor instructions for workflow and docs changes.

This repository has no application source tree or dedicated test directory. Keep changes focused on workflow behavior and documentation accuracy.

## Build, Test, and Development Commands

There is no local build pipeline. Useful contributor commands are:

- `git status`: confirm the working tree before and after edits.
- `git diff`: review YAML, shell, and Markdown changes before committing.
- `sed -n '1,220p' .github/workflows/rule-sync.yml`: inspect the workflow logic.
- `gh workflow run rule-sync.yml`: manually trigger the sync when `gh` is configured.

Validate changes in GitHub Actions instead of assuming local shell snippets are enough.

## Coding Style & Naming Conventions

- Use 2-space indentation in YAML.
- Keep workflow step names short and action-oriented, for example `Download SRS files`.
- In shell blocks, quote variables as `"${VAR}"`, keep commands simple, and avoid printing authenticated URLs or tokens.
- Preserve clear file names that match upstream artifacts, such as `adblock_reject.srs`.
- Update docs in the same change whenever URLs, secret names, schedule, or destination paths change.

## Testing Guidelines

There is no automated test suite yet. Validation is workflow-based:

- Confirm download URLs still resolve.
- Confirm the sync target remains `gitea-repo/rules/` unless intentionally changed.
- Confirm the fallback path for missing repositories still works: clone failure should trigger a Gitea API create call.
- Confirm no secrets are printed in logs.
- Include the relevant GitHub Actions run result in the PR description when behavior changes.

## Commit & Pull Request Guidelines

The history is minimal and includes plain-language commits plus automation-generated commits such as `chore: update srs rule files [...]`. For manual changes, prefer short subjects like `ci: handle Gitea API errors` or `docs: update sync path`.

PRs should include the purpose of the change, impacted files, any required secret or config updates, and the validation performed. Screenshots are unnecessary unless GitHub UI settings are part of the change.

## Security & Configuration Tips

Never commit tokens, authenticated clone URLs, or real server endpoints. Keep secret names stable: `GITEA_URL`, `GITEA_REPO`, `GITEA_TOKEN`, and `GITEA_BRANCH`. If you change repository creation or push behavior, review both `git clone` and `curl /api/v1/user/repos` paths for secret exposure.
