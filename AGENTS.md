# Repository Guidelines

## Project Structure & Module Organization

- `.github/workflows/rule-sync.yml`: the core automation. It downloads upstream `.srs` files and pushes updates to the target Gitea repository.
- `README.md`: project overview, setup notes, and operator-facing instructions.
- `AGENTS.md`: contributor guidance for repository changes.

This repository currently has no application source tree, package manifest, or dedicated test directory. Keep changes focused on workflow behavior and documentation accuracy.

## Build, Test, and Development Commands

There is no local build pipeline. Useful contributor commands are:

- `git status`: confirm the working tree before and after edits.
- `git diff`: review YAML, shell, and Markdown changes before committing.
- `sed -n '1,220p' .github/workflows/rule-sync.yml`: inspect the current workflow logic.
- `gh workflow run rule-sync.yml`: manually trigger the sync workflow on GitHub when `gh` is configured.

When changing sync behavior, validate the GitHub Actions run rather than assuming local edits are sufficient.

## Coding Style & Naming Conventions

- Use 2-space indentation in YAML.
- Keep workflow step names short and action-oriented, for example `Download SRS files`.
- In shell blocks, quote variables as `"${VAR}"`, keep commands simple, and avoid leaking secrets in logs.
- Preserve clear file names that match upstream artifacts, such as `adblock_reject.srs`.
- Update documentation in the same change whenever URLs, secret names, or destination paths change.

## Testing Guidelines

There is no automated test suite yet. Validation is workflow-based:

- Confirm download URLs still resolve.
- Confirm the target copy path in the workflow matches the intended Gitea layout.
- Confirm no secrets are printed in logs.
- Include the relevant GitHub Actions run result in the PR description when behavior changes.

## Commit & Pull Request Guidelines

The history is minimal and includes both plain-language commits and automation-generated `chore:` commits. For manual changes, use a short imperative subject and keep it specific, for example `docs: clarify Gitea secret setup`.

PRs should include the purpose of the change, impacted files, any required secret or config updates, and the validation performed. Screenshots are unnecessary unless GitHub UI settings are part of the change.

## Security & Configuration Tips

Never commit tokens, authenticated clone URLs, or real server endpoints. Keep secret names stable: `GITEA_URL`, `GITEA_REPO`, `GITEA_TOKEN`, and `GITEA_BRANCH`.
