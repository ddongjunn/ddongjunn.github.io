# Resume Review Panel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Install and verify a personal Codex skill that runs three independent resume reviewers followed by one final synthesis reviewer.

**Architecture:** A single `SKILL.md` defines orchestration and fixed report schemas. Three reviewers run in parallel within the four-slot limit, then one synthesizer runs after they finish. `agents/openai.yaml` provides discoverable UI metadata.

**Tech Stack:** Codex personal skills, collaboration subagents, Markdown, YAML

---

### Task 1: Record Baseline Behavior

**Files:**
- Read: `_tabs/about.md`
- Read: `resume/v1.md`

- [ ] **Step 1: Run a comparison request without the new skill**

Ask a fresh subagent to compare the two resume files through multiple perspectives and return one report, without providing the proposed workflow.

Expected: At least one required behavior is missing or inconsistent, such as three independent reports, a separate synthesizer, a shared rubric, or one compact final output.

- [ ] **Step 2: Record the missing behavior**

Keep the result in the active session as the RED evidence. Do not create repository artifacts.

### Task 2: Initialize the Personal Skill

**Files:**
- Create: `/Users/djlee/.codex/skills/resume-review-panel/SKILL.md`
- Create: `/Users/djlee/.codex/skills/resume-review-panel/agents/openai.yaml`

- [ ] **Step 1: Initialize the skill**

Run:

```bash
python3 /Users/djlee/.codex/skills/.system/skill-creator/scripts/init_skill.py resume-review-panel --path /Users/djlee/.codex/skills --interface display_name="Resume Review Panel" --interface short_description="Independent resume scoring with one final synthesis" --interface default_prompt="Use $resume-review-panel to compare my current resume with a previous version."
```

Expected: A new personal skill directory with `SKILL.md` and `agents/openai.yaml`.

- [ ] **Step 2: Replace `SKILL.md` with the minimal workflow**

Write instructions that require:

```text
resolve source files
spawn product, technical, and writing reviewers in parallel
collect the same score schema from each reviewer
spawn one synthesizer only after all reviewers finish
return one compact report
```

The workflow must be read-only by default, use actual source files, retry a failed reviewer once, and disclose a missing perspective after a second failure.

### Task 3: Validate Structure and Content

**Files:**
- Validate: `/Users/djlee/.codex/skills/resume-review-panel/SKILL.md`
- Validate: `/Users/djlee/.codex/skills/resume-review-panel/agents/openai.yaml`

- [ ] **Step 1: Run the official validator**

Run:

```bash
python3 /Users/djlee/.codex/skills/.system/skill-creator/scripts/quick_validate.py /Users/djlee/.codex/skills/resume-review-panel
```

Expected: `Skill is valid!`

- [ ] **Step 2: Inspect generated metadata**

Confirm the default prompt names `$resume-review-panel` and no placeholder text remains.

### Task 4: Forward Test the Workflow

**Files:**
- Read: `_tabs/about.md`
- Read: `resume/v1.md`
- Read: `/Users/djlee/.codex/skills/resume-review-panel/SKILL.md`

- [ ] **Step 1: Invoke the skill on the real comparison**

Use the request:

```text
Use $resume-review-panel to compare _tabs/about.md with resume/v1.md, focusing on the Ansible project.
```

- [ ] **Step 2: Verify required behavior**

Confirm all of the following:

```text
three independent reviewers ran
all reviewers used the same 0-to-10 schema
one separate synthesizer ran after reviewers completed
the final output contained one score table and resolved disagreements
no resume files were edited
```

- [ ] **Step 3: Re-run validation after any correction**

Run the official validator again and require `Skill is valid!` before completion.
