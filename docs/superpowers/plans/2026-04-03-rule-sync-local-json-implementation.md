# Rule Sync Local JSON Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the sync workflow so it compiles `config/*.json` into `.srs`, logs per-file compile failures as warnings with concrete error output, and still pushes all available `.srs` files to Gitea.

**Architecture:** Keep the existing single-job workflow and single staging directory `rules/`. Add a pinned `sing-box` install step plus a compile loop that soft-fails per JSON file, then preserve the existing `rules/*.srs` push flow. Update README so the operator-facing description matches the workflow behavior.

**Tech Stack:** GitHub Actions YAML, bash, sing-box, Markdown

---

### Task 1: Update Workflow Compilation and Logging

**Files:**
- Modify: `.github/workflows/rule-sync.yml`
- Reference: `config/*.json`

- [ ] **Step 1: Edit the workflow to install `sing-box` and compile local JSON rule sets**

```yaml
      - name: Install sing-box
        env:
          SING_BOX_VERSION: 1.12.20
        run: |
          mkdir -p "${HOME}/.local/bin" /tmp/sing-box
          curl -fsSL -o /tmp/sing-box.tar.gz \
            "https://github.com/SagerNet/sing-box/releases/download/v${SING_BOX_VERSION}/sing-box-${SING_BOX_VERSION}-linux-amd64.tar.gz"
          tar -xzf /tmp/sing-box.tar.gz -C /tmp/sing-box --strip-components=1
          install /tmp/sing-box/sing-box "${HOME}/.local/bin/sing-box"
          echo "${HOME}/.local/bin" >> "${GITHUB_PATH}"
          sing-box version

      - name: Compile local JSON rule sets
        shell: bash
        run: |
          if [ ! -d config ]; then
            echo "No config directory found, skip local JSON compilation."
            exit 0
          fi

          shopt -s nullglob
          json_files=(config/*.json)
          if [ "${#json_files[@]}" -eq 0 ]; then
            echo "No local JSON rule sets found, skip local compilation."
            exit 0
          fi

          failed_files=()
          for json_file in "${json_files[@]}"; do
            output_file="rules/$(basename "${json_file}" .json).srs"
            error_log="/tmp/$(basename "${json_file}" .json)-compile.log"

            echo "Compiling ${json_file} -> ${output_file}"
            if sing-box rule-set compile --output "${output_file}" "${json_file}" >"${error_log}" 2>&1; then
              echo "Compiled ${json_file} successfully."
            else
              failed_files+=("${json_file}")
              echo "::warning file=${json_file}::Failed to compile ${json_file}; skipped this file."
              echo "Compile error for ${json_file}:"
              cat "${error_log}"
              rm -f "${output_file}"
            fi
          done

          if [ "${#failed_files[@]}" -gt 0 ]; then
            echo "::warning::Local JSON compilation completed with failures: ${failed_files[*]}"
          fi

          ls -lh rules/
```

- [ ] **Step 2: Review the workflow structure and ensure the push step still copies only `.srs` files**

Run: `sed -n '1,260p' .github/workflows/rule-sync.yml`
Expected: the workflow contains `Install sing-box`, `Compile local JSON rule sets`, and the push step still uses `cp rules/*.srs gitea-repo/rules/`

- [ ] **Step 3: Verify the workflow YAML parses**

Run: `ruby -e 'require "yaml"; YAML.load_file(".github/workflows/rule-sync.yml"); puts "YAML OK"'`
Expected: `YAML OK`

### Task 2: Update Operator Documentation

**Files:**
- Modify: `README.md`
- Reference: `.github/workflows/rule-sync.yml`

- [ ] **Step 1: Update README to describe local JSON compilation, warning-only failures, and the real target path**

```markdown
- 支持将 `config/*.json` 使用 `sing-box` 编译为 `.srs`
- 单个本地 JSON 编译失败时只输出 warning，不中断整体同步
- 最终只会将 `.srs` 文件推送到目标 Gitea 仓库的 `rules/` 目录
```

- [ ] **Step 2: Correct the cron explanation so it matches the workflow**

```markdown
说明：GitHub Actions 的 `cron` 使用 UTC 时区，当前配置 `0 12 * * 4` 对应北京时间每周四 20:00。
```

- [ ] **Step 3: Verify the README reflects the new behavior**

Run: `sed -n '1,260p' README.md`
Expected: the README mentions `config/*.json` compilation, warning-only compile failures, `rules/` as the destination, and the corrected cron explanation

### Task 3: Final Review

**Files:**
- Review: `.github/workflows/rule-sync.yml`
- Review: `README.md`

- [ ] **Step 1: Inspect the final diff**

Run: `git diff -- .github/workflows/rule-sync.yml README.md`
Expected: only the intended workflow and documentation changes appear

- [ ] **Step 2: Check worktree state**

Run: `git status --short`
Expected: modified workflow/docs files are visible, and no unexpected tracked-file regressions appear
