---
description: 'Codex 审查 + Claude 综合（独立工具，随时可用）'
---
<!-- CCG:SPEC:REVIEW:START -->
**Core Philosophy**
- Codex 深度代码审查 + Claude 架构综合评估
- Critical findings MUST be addressed before proceeding.
- Review validates implementation against spec constraints and code quality.
- This is an independent review tool—can be used anytime.

**Guardrails**
- **MANDATORY**: Codex must complete review, Claude synthesizes and supplements.
- Review scope is strictly limited to the proposal's changes—no scope creep.

**Steps**
1. **Select Proposal**
   - Run `/opsx:list` to display Active Changes.
   - Confirm with user which proposal ID to review.
   - Run `/opsx:show <proposal_id>` to load spec and tasks.

2. **Collect Implementation Artifacts**
   - Identify all files modified by this proposal.
   - Use `git diff` or `/opsx:diff <proposal_id>` to get change summary.
   - Load relevant spec constraints from `openspec/changes/<id>/specs/`.

3. **Codex Code Review**
   Launch Codex review with `run_in_background: true`:

   ```
   Bash({
     command: "/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend codex - \"$PWD\" <<'EOF'\nReview proposal <proposal_id> implementation:\n\n## Review Dimensions\n1. **Spec Compliance**: Verify ALL constraints from spec are satisfied\n2. **Logic Correctness**: Edge cases, error handling, algorithm correctness\n3. **Security**: Injection vulnerabilities, auth checks, XSS, CSRF\n4. **Pattern Consistency**: Naming conventions, code style, project patterns\n5. **Maintainability**: Readability, complexity, documentation adequacy\n6. **Regression Risk**: Interface compatibility, type safety, breaking changes\n\n## Output Format (JSON)\n{\n  \"findings\": [\n    {\n      \"severity\": \"Critical|Warning|Info\",\n      \"dimension\": \"spec|logic|security|pattern|maintainability|regression\",\n      \"file\": \"path/to/file\",\n      \"line\": 42,\n      \"description\": \"What is wrong\",\n      \"fix_suggestion\": \"How to fix\"\n    }\n  ],\n  \"passed_checks\": [\"List of verified aspects\"],\n  \"summary\": \"Overall assessment\"\n}\nEOF",
     run_in_background: true,
     timeout: 600000,
     description: "Codex code review"
   })
   ```

   Wait for result:
   ```
   TaskOutput({ task_id: "<codex_task_id>", block: true, timeout: 600000 })
   ```

4. **Claude Synthesis**
   Based on Codex review:
   - Verify findings accuracy
   - Add architecture/design level concerns
   - Deduplicate and classify by severity:
     * **Critical**: Spec violation, security vulnerability, breaking change → MUST fix
     * **Warning**: Pattern deviation, maintainability concern → SHOULD fix
     * **Info**: Minor improvement suggestion → MAY fix

5. **Present Review Report**
   ```
   ## Review Report: <proposal_id>

   ### Critical (X issues) - MUST FIX
   - [ ] [SPEC] file.ts:42 - Constraint X violated: description
   - [ ] [SEC] api.ts:15 - Injection vulnerability

   ### Warning (Y issues) - SHOULD FIX
   - [ ] [PATTERN] utils.ts:88 - Inconsistent naming convention

   ### Info (Z issues) - MAY FIX
   - [ ] [MAINT] helper.ts:20 - Consider extracting to separate function

   ### Passed Checks
   - ✅ Security: No critical vulnerabilities found
   - ✅ Logic: Edge cases handled correctly
   ```

6. **Decision Gate**
   - **If Critical > 0**:
     * Present findings to user.
     * Ask: "Fix now or return to `/ccg:spec-impl` to address?"
     * Do NOT allow archiving.

   - **If Critical = 0**:
     * Ask user: "All critical checks passed. Proceed to archive?"
     * If Warning > 0, recommend addressing before archive.

7. **Iterative Fix Mode**
   - If user chooses "Fix now" for Critical issues:
     * Claude directly fixes the code
     * Re-run Codex review on changed files
     * Repeat until Critical = 0

**Exit Criteria**
Review is complete when:
- [ ] Codex review completed
- [ ] Claude synthesis done
- [ ] Zero Critical issues remain (fixed or user-acknowledged)
- [ ] User decision captured (archive / return to impl / defer)

**Reference**
- View proposal: `/opsx:show <id>`
- Check spec constraints: `rg -n "CONSTRAINT:|MUST|INVARIANT:" openspec/changes/<id>/specs/`
- View implementation diff: `/opsx:diff <id>` or `git diff`
<!-- CCG:SPEC:REVIEW:END -->
