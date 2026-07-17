# Resume Review Panel Design

## Goal

Create a personal Codex skill that reviews one resume or compares a current resume with a baseline through three independent reviewers and one final synthesizer.

## Location and Trigger

- Install at `~/.codex/skills/resume-review-panel`.
- Trigger on requests such as `이력서 패널 리뷰`, `이력서 에이전트 점수`, or `현재 이력서와 이전 버전을 비교 평가`.

## Workflow

1. Resolve the target resume, optional baseline, and any supporting project files.
2. Spawn three reviewers in parallel with isolated task context:
   - Product and recruiting reviewer
   - Technical interview reviewer
   - Resume writing reviewer
3. Require each reviewer to return:
   - Overall score out of 10
   - Section scores when applicable
   - Strengths
   - Deductions
   - Submission readiness
   - One highest-impact improvement
4. Wait for all three reviewers to complete.
5. Spawn one final synthesizer with the source paths and raw reviewer reports.
6. Return one compact report containing:
   - Score table
   - Shared findings
   - Conflicting findings and the synthesizer's decision
   - Prioritized improvements
   - Final recommendation or copy when requested

## Rules

- Reviewers must not see one another's reports.
- Reviewers must inspect the actual source files instead of relying on conversation summaries.
- The synthesizer must not average blindly. It must resolve conflicts against source evidence.
- Keep the workflow read-only unless the user explicitly asks to edit files.
- Do not invent team adoption, usage metrics, technical decisions, or outcomes absent from the source.
- When comparing versions, score both with the same rubric and state regressions as well as improvements.
- If a reviewer fails, retry once. If it still fails, synthesize the available reviews and disclose the missing perspective.

## Output Shape

The final response should use a single score table followed by concise sections for shared findings, disagreements, priorities, and the final recommendation. Do not expose orchestration logs or make the user read four separate reports.

## Validation

- Baseline: run a resume comparison request without the skill and record inconsistent or missing behavior.
- Forward test: run the same request with the skill and verify three independent reports, one synthesizer, identical scoring dimensions, source inspection, and no file edits.
- Validate the skill structure with the official skill validator.
