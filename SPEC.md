# Readable Ralph — Feature Specification

> **Product goal**: A rich viewer for gulf-loop execution logs and artifacts.
> Non-technical users must be able to trace, per-iteration, exactly what the AI did.

**Stack**: Tauri 2 · React 18 · TypeScript · Zustand · Rust (`portable_pty`, `notify`)
**Design system**: Dark glassmorphism, sharp corners (`--radius: 0px`), accent green `#10d9a0`
**Source of truth for implementation**: `/non-dev-readable-ralph/src` and `src-tauri`

---

## Table of Contents

1. [Existing Architecture Reference](#1-existing-architecture-reference)
2. [File Watch & Event Contracts](#2-file-watch--event-contracts)
3. [Data Schemas & Parser Contracts](#3-data-schemas--parser-contracts)
4. [Type Definitions (Authoritative)](#4-type-definitions-authoritative)
5. [Store Shape](#5-store-shape)
6. [State Machine](#6-state-machine)
7. [Feature Specifications](#7-feature-specifications)
8. [UI / Layout Specifications](#8-ui--layout-specifications)
9. [File Scope](#9-file-scope)
10. [Acceptance Criteria (Binary Pass/Fail)](#10-acceptance-criteria-binary-passfail)
11. [Future Work](#11-future-work)

---

## 1. Existing Architecture Reference

### 1.1 Frontend Routing (App.tsx)

```typescript
const loopIsActive = loopState !== null && loopState.phase !== 'idle'
const projectIsStarting = activeProject?.status === 'running' && loopState === null

return showWizard       ? <GoalWizard />
     : loopIsActive     ? <LoopMonitor />
     : projectIsStarting? <StartingScreen />
     : activeProject    ? <QuickRestartView />
     : <EmptyState />
```

### 1.2 Tauri Command Reference (all existing commands)

| Command | Signature | Notes |
|---------|-----------|-------|
| `check_claude_version` | `() -> String` | |
| `check_gulf_loop_installed` | `() -> bool` | |
| `list_projects` | `() -> Vec<Project>` | |
| `create_project` | `(name, path, projectType) -> Result<Project>` | |
| `delete_project` | `(id) -> Result<()>` | |
| `update_project_status` | `(id, status) -> Result<()>` | |
| `detect_project_type` | `(path) -> String` | `webapp\|cli\|script\|other` |
| `get_settings` | `() -> RalphSettings` | |
| `save_settings` | `(settings) -> Result<()>` | |
| `spawn_claude` | `(app, projectId, projectPath, cols, rows) -> Result<()>` | standalone spawn, no command |
| `spawn_and_send` | `(app, projectId, projectPath, command, cols, rows) -> Result<()>` | spawn + `tokio::sleep(8s)` + write; returns after send |
| `send_command` | `(projectId, data) -> Result<()>` | raw PTY write |
| `send_command_enter` | `(projectId, data) -> Result<()>` | `write_all("{data}\r")` — carriage return only, no `\n` (see §7.11.5) |
| `kill_claude` | `(projectId) -> Result<()>` | |
| `kill_all_sessions` | `() -> Result<()>` | |
| `force_reset_session` | `(projectId, projectPath) -> Result<()>` | kills PTY + deletes state file |
| `resize_session` | `(projectId, cols, rows) -> Result<()>` | |
| `is_session_active` | `(projectId) -> bool` | |
| `start_watcher` | `(app, projectPath) -> Result<()>` | |
| `stop_watcher` | `() -> Result<()>` | |
| `get_git_log` | `(projectPath, limit) -> Vec<GitCommit>` | |
| `get_git_diff` | `(projectPath, commitHash) -> String` | truncated to 8000 chars |
| `save_session` | `(projectPath, sessionJson) -> Result<()>` | writes `.ralph/session.json` |
| `load_session` | `(projectPath) -> Option<String>` | |
| `archive_session` | `(projectPath) -> Result<()>` | moves to `.ralph/sessions/{ts}.json` |
| `list_sessions` | `(projectPath) -> Vec<String>` | newest first |
| `write_file` | `(path, content) -> Result<()>` | |
| `read_file` | `(path) -> Result<String>` | |
| `file_exists` | `(path) -> bool` | |

### 1.3 CSS Variables (Design System)

```css
/* Backgrounds */
--bg-base: #0a0a0a;       --bg-0: #111111;
--bg-1: #161616;          --bg-2: #1a1a1a;
--bg-3: #202020;          --bg-hover: rgba(255,255,255,0.06);

/* Glass */
--glass-bg: rgba(12,12,20,0.70);
--glass-border: rgba(255,255,255,0.07);
--glass-blur: blur(22px) saturate(160%);
--glass-shadow: 0 8px 32px rgba(0,0,0,0.55), 0 2px 8px rgba(0,0,0,0.35);

/* Borders */
--border: rgba(255,255,255,0.08);
--border-mid: rgba(255,255,255,0.13);

/* Text */
--text-0: #eeeef5;    --text-1: #a8a8c0;
--text-2: #686880;    --text-3: #3a3a52;

/* Accent */
--accent: #10d9a0;    --accent-dim: #059669;
--accent-glow: rgba(16,217,160,0.13);
--accent-glow-strong: rgba(16,217,160,0.26);

/* Status */
--green: #4ade80;     --red: #f87171;
--yellow: #fbbf24;    --cyan: #22d3ee;

/* Typography */
--font-mono: 'JetBrains Mono', 'Fira Code', monospace;
--font-sans: -apple-system, 'SF Pro Display', system-ui, sans-serif;

/* Shape (all sharp) */
--radius: 0px;    --radius-lg: 0px;    --radius-xl: 0px;

/* Layout */
--sidebar-width: 240px;     --right-panel-width: 280px;
--titlebar-height: 36px;    --statusbar-height: 28px;

/* Animation */
--t-fast: 0.12s ease;    --t-mid: 0.2s ease;
--t-slow: 0.35s cubic-bezier(0.4,0,0.2,1);
```

---

## 2. File Watch & Event Contracts

`start_watcher(app, projectPath)` monitors 5 files. On start, it **immediately emits the current state** of all existing files.

| File | Event Name | Payload Type |
|------|-----------|--------------|
| `.claude/gulf-loop.local.md` | `loop-state-changed` | `{ content: string \| null, deleted: boolean }` |
| `progress.txt` | `progress-changed` | `{ content: string \| null }` |
| `JUDGE_FEEDBACK.md` | `judge-feedback-changed` | `{ content: string \| null }` |
| `JUDGE_EVOLUTION.md` | `judge-evolution-changed` | `{ content: string \| null, deleted: boolean }` |
| `RUBRIC.md` | `rubric-changed` | `{ content: string \| null, deleted: boolean }` |

**Watcher contracts:**
- File changes are debounced with **100ms delay** before reading, to avoid partial-write reads.
- `deleted: true` means the file no longer exists.
- `content: null` means the file was deleted or could not be read.
- Receiving `content: null` must set the corresponding store field to `null`, not be silently ignored.
- The frontend must handle all 5 events even when the file content is invalid; the app must never crash on malformed input.

**PTY Events (from `process/mod.rs`):**

| Event | Payload | Trigger |
|-------|---------|---------|
| `claude-output` | `{ project_id: string, data: string }` | Per chunk of PTY stdout |
| `claude-auth-required` | `{}` | Auth prompt detected in PTY output |
| `claude-session-dead` | `{ project_id: string }` | PTY reader thread exits |

---

## 3. Data Schemas & Parser Contracts

### 3.1 `.claude/gulf-loop.local.md`

**Location:** `{project_root}/.claude/gulf-loop.local.md`

**File structure:**
```
---
[YAML frontmatter]
---
[Original prompt body — everything after second ---]
```

**YAML field schema:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `active` | `boolean` | `false` | `true` = loop is running |
| `iteration` | `number` | `1` | Current iteration (1-based) |
| `max_iterations` | `number` | `50` | Maximum iterations allowed |
| `completion_promise` | `string` | `"COMPLETE"` | Tag that signals completion |
| `judge_enabled` | `boolean` | `false` | Judge mode on/off |
| `consecutive_rejections` | `number` | `0` | Consecutive rejection count |
| `hitl_threshold` | `number` | `5` | Rejections before HITL pause |
| `autonomous` | `boolean` | `false` | Autonomous branch mode |
| `branch` | `string` | `""` | Feature branch name |
| `base_branch` | `string` | `"main"` | Merge target branch |
| `milestone_every` | `number` | `0` | `0` = disabled; N = pause every N rounds |
| `worktree_path` | `string` | `""` | Parallel mode worktree path |
| `structured_memory` | `boolean` | `false` | Use `.claude/memory/` wiki |
| `force_max` | `boolean` | `false` | Ignore completion signal |
| `pause_reason` | `string` | `""` | `"milestone"` \| `"hitl"` \| `""` |

**Parser contract (`state-parser.ts` → `parseLoopState`):**

```typescript
function parseLoopState(content: string | null | undefined): LoopState | null
```

- Returns `null` if `content` is null, undefined, or empty.
- Returns `null` if YAML frontmatter cannot be parsed (malformed `---` block).
- All missing YAML fields → schema defaults above.
- `iteration` non-numeric → fallback `1`.
- `max_iterations` ≤ 0 → fallback `50`.
- Original prompt: everything after the closing `---`, trimmed. If no body exists, `originalPrompt: null`.
- **Phase derivation logic:**

  ```
  active = true  AND  pause_reason = ''       → phase = 'running'
  active = true  AND  pause_reason ≠ ''       → phase = 'paused'
  active = false AND  pause_reason = 'hitl'   → phase = 'paused'
  active = false AND  pause_reason = 'milestone' → phase = 'paused'
  active = false AND  pause_reason = ''       → phase = 'completed'
  ```

- `loopMode` field: `autonomous: true` → `'autonomous'`; `judge_enabled: true` → `'judge'`; else `'basic'`.

---

### 3.2 `progress.txt`

**Location:** `{project_root}/progress.txt` — **NOT** inside `.claude/`.

**Standard format (all iterations):**

```
ORIGINAL_GOAL: [original user goal — written in iteration 1, never changes]
ITERATION: [N]

COMPLETED:
- [completed task description]

DECISIONS:
- chose: [X], rejected: [Y], reason: [reason], revisit_if: [condition]

UNCERTAINTIES:
- [uncertain item]

REMAINING_GAP:
- [remaining work item]

CONFIDENCE: [30-100]

NEXT_STEP: [next planned action]
```

**Iteration 1 additional fields:**

```
STRENGTHS:
- [strength to preserve]

RISKS:
- [major risk]

GAPS:
- [unknown item]

APPROACHES_CONSIDERED:
- [approach A]: [rejection reason]

APPROACH:
[chosen approach description — minimum 50 characters]
```

**Parser contract (`progress-parser.ts` → `parseProgress`):**

```typescript
function parseProgress(content: string | null | undefined): ProgressData | null
```

- Returns `null` if `content` is null, undefined, or empty string.
- Returns a `ProgressData` with all defaults if content is non-empty but no sections match. Never throws.
- Section headers are **case-sensitive** (`COMPLETED:`, not `completed:`).
- Items under each section start with `- ` (dash + space). Lines not starting with `- ` are ignored.

| Field | Extraction Rule | Default |
|-------|----------------|---------|
| `originalGoal` | Text on same line as `ORIGINAL_GOAL:` | `null` |
| `iterationNumber` | Integer after `ITERATION:` | `null` |
| `completed` | Array of `- ` items under `COMPLETED:` | `[]` |
| `decisions` | Array parsed from `DECISIONS:` section | `[]` |
| `uncertainties` | Array of `- ` items under `UNCERTAINTIES:` | `[]` |
| `remainingGap` | Array of `- ` items under `REMAINING_GAP:` | `[]` |
| `confidence` | Integer after `CONFIDENCE:`, clamped 30–100 | `null` |
| `nextStep` | Text on same line as `NEXT_STEP:` | `null` |

**`DECISIONS` line parsing:**

```typescript
const decisionRegex =
  /chose:\s*([^,]+)(?:,\s*rejected:\s*([^,]+))?(?:,\s*reason:\s*([^,]+))?(?:,\s*revisit_if:\s*(.+))?/i;
```

- Line without `chose:` → skip.
- `rejected`, `reason`, `revisit_if` are optional → `null` if absent.
- Trailing whitespace trimmed from all extracted values.

**Edge cases:**
- `CONFIDENCE: abc` → `confidence: null`
- `CONFIDENCE: 150` → `confidence: 100` (clamp)
- `CONFIDENCE: 10` → `confidence: 30` (clamp)
- Duplicate section headers → first occurrence wins.
- `- ` items before the first section header → ignored.

---

### 3.3 `JUDGE_FEEDBACK.md`

**Format:**

```markdown
---
## Iteration 7 — REJECTED (3 consecutive) — 2026-02-27 14:32:01

[Summary line — first non-empty line after header]

[Additional context]
---
```

**Parser contract (`parseJudgeFeedback`):**

```typescript
function parseJudgeFeedback(content: string | null | undefined): JudgeFeedbackEntry[]
```

- Returns `[]` if content is null/undefined/empty.
- Blocks separated by `---` delimiters.
- A file without `---` delimiters is treated as a single block.

| Field | Extraction Rule | Fallback |
|-------|----------------|---------|
| `id` | `"feedback-${index}"` (0-indexed) | required |
| `iteration` | `/Iteration\s+(\d+)/i` → parseInt | `null` |
| `approved` | header contains `APPROVED` (case-insensitive) | `false` |
| `consecutive` | `/\((\d+)\s+consecutive\)/i` → parseInt | `0` |
| `timestamp` | last `—`-separated segment in `## ` header line | `""` |
| `summary` | first non-empty line after the `## ` header | `""` |
| `rawText` | full block text | required |
| `rejectionType` | `"autocheck"` if block contains `FAILED:` section; `"judge"` otherwise | `null` if approved |
| `autochecksPassed` | lines starting with `✓` | `[]` |
| `autochecksFailed` | lines starting with `✗` | `[]` |

---

### 3.4 `JUDGE_EVOLUTION.md`

**Format (one entry per line):**

```
[iter 3] REJECTED — silent catch blocks contain errors that should propagate
[iter 5] APPROVED — all async handlers use consistent error structure
```

**Parser (inline, no separate function needed):**

```typescript
const lineRegex = /^\[iter\s+(\d+)\]\s+(APPROVED|REJECTED)\s*—\s*(.+)$/i;
```

- Lines not matching the regex → silently skipped.
- Parsed fields: `iteration: number`, `approved: boolean`, `principle: string`.
- Display order: newest (highest `iteration`) first.

---

## 4. Type Definitions (Authoritative)

These types are the canonical source of truth. All new code must conform to them.

```typescript
// src/types/loop.ts

export type LoopPhase = 'idle' | 'running' | 'paused' | 'completed' | 'cancelled';
export type LoopMode = 'basic' | 'judge' | 'autonomous' | 'parallel';
export type ActivityEventType =
  | 'loop-started'
  | 'iteration-complete'
  | 'iteration-failed'
  | 'loop-paused-hitl'
  | 'loop-paused-milestone'
  | 'loop-resumed'
  | 'loop-completed'
  | 'loop-cancelled'
  | 'progress-updated';

export interface LoopState {
  active: boolean;
  phase: LoopPhase;
  iteration: number;
  maxIterations: number;
  completionPromise: string;
  judgeEnabled: boolean;
  consecutiveRejections: number;
  hitlThreshold: number;
  loopMode: LoopMode;
  autonomous: boolean;
  branch: string | null;
  baseBranch: string | null;
  milestoneEvery: number;
  worktreePath: string;
  structuredMemory: boolean;
  forceMax: boolean;
  pauseReason: 'milestone' | 'hitl' | '';
  startedAt: string | null;
  originalPrompt: string | null;
}

export interface DecisionItem {
  chose: string;
  rejected: string | null;
  reason: string | null;
  revisitIf: string | null;
}

export interface ProgressData {
  completed: string[];
  remainingGap: string[];
  confidence: number | null;   // null = not parseable; always 30–100 when present
  decisions: DecisionItem[];
  uncertainties: string[];
  nextStep: string | null;
  originalGoal: string | null;
  iterationNumber: number | null;
}

export interface ProgressSnapshot {
  iteration: number;
  completed: string[];
  decisions: DecisionItem[];
  uncertainties: string[];
  remainingGap: string[];
  confidence: number | null;
  nextStep: string | null;
}

export interface GitCommit {
  hash: string;
  shortHash: string;
  message: string;
  authorEmail: string;
  timestamp: string;
  filesChanged: string[];
  stats: string;
}

export interface JudgeFeedbackEntry {
  id: string;
  iteration: number | null;
  approved: boolean;
  consecutive: number;
  timestamp: string;
  summary: string;
  rawText: string;
  rejectionType: 'autocheck' | 'judge' | null;
  autochecksPassed: string[];
  autochecksFailed: string[];
}

export interface IterationItem {
  iteration: number;
  status: 'completed' | 'active' | 'pending' | 'failed';
  label: string;              // "Round N"
  sublabel?: string;          // progress.nextStep value
  progressSnapshot?: ProgressSnapshot;
  gitCommits?: GitCommit[];
}

export interface ActivityEvent {
  id: string;
  type: ActivityEventType;
  iteration: number;
  timestamp: number;          // Date.now()
  label: string;              // Human-readable description
  detail?: string;            // Optional secondary description
}
```

```typescript
// src/types/project.ts (existing — no changes required)

export type ProjectStatus = 'idle' | 'running' | 'paused' | 'completed' | 'cancelled';
export type ProjectType = 'webapp' | 'cli' | 'api' | 'script' | 'desktop' | 'other';

export interface RalphProject {
  id: string;
  name: string;
  path: string;
  projectType: ProjectType;
  status: ProjectStatus;
  createdAt: string;
  lastActiveAt: string;
  totalIterations: number;
  totalTimeMs: number;
  estimatedCostKrw: number;
}

export interface RalphSettings {
  theme: string;
  locale: 'ko' | 'en';
  advancedMode: boolean;
}
```

---

## 5. Store Shape

### 5.1 LoopSlice (additions to existing slice)

New fields added to existing `LoopSlice`:

```typescript
// NEW fields to add to loopSlice.ts
interface LoopSliceAdditions {
  rawProgressContent: string | null;       // Raw text of progress.txt (for dev mode viewer)
  activityEvents: ActivityEvent[];         // Chronological event log
  addActivityEvent: (event: ActivityEvent) => void;
  // setProgress signature change:
  setProgress: (progress: ProgressData | null, raw: string | null) => void;  // was setProgress(ProgressData | null)
}
```

Full loop slice shape after additions:

```typescript
interface LoopSlice {
  // Existing
  loopState: LoopState | null;
  progress: ProgressData | null;
  judgeFeedback: JudgeFeedbackEntry[];
  iterations: IterationItem[];
  loopStartTimeMs: number | null;
  rubricsContent: string | null;
  judgeEvolution: string | null;

  // New
  rawProgressContent: string | null;
  activityEvents: ActivityEvent[];

  // Existing actions
  setLoopState: (state: LoopState | null) => void;
  setJudgeFeedback: (entries: JudgeFeedbackEntry[]) => void;
  addJudgeFeedback: (entry: JudgeFeedbackEntry) => void;
  updateIteration: (item: IterationItem) => void;
  setIterations: (items: IterationItem[]) => void;
  setLoopStartTime: (ms: number | null) => void;
  resetLoop: () => void;
  setRubricsContent: (content: string | null) => void;
  setJudgeEvolution: (content: string | null) => void;
  setIterationGitCommits: (iteration: number, commits: GitCommit[]) => void;

  // Updated action (signature change)
  setProgress: (progress: ProgressData | null, raw: string | null) => void;

  // New action
  addActivityEvent: (event: ActivityEvent) => void;
}
```

### 5.2 SettingsSlice (existing — no changes)

```typescript
interface SettingsSlice {
  locale: 'ko' | 'en';
  theme: string;
  advancedMode: boolean;
  onboardingDone: boolean;
  showTerminal: boolean;
  showRightPanel: boolean;
  authRequired: boolean;
  rightTab: 'activity' | 'memory' | 'settings' | null;

  setLocale, setTheme, setAdvancedMode, setOnboardingDone,
  setShowTerminal, setShowRightPanel, setAuthRequired, setRightTab
}
```

### 5.3 ActivityEvent Generation Rules

These events must be generated **inside `setLoopState`** when the state transitions:

| Condition | Event type |
|-----------|-----------|
| `prev === null` AND `next.phase === 'running'` | `loop-started` |
| `prev` exists AND `next.iteration > prev.iteration` | `iteration-complete` |
| `prev.phase !== 'paused'` AND `next.phase === 'paused'` AND `next.pauseReason === 'hitl'` | `loop-paused-hitl` |
| `prev.phase !== 'paused'` AND `next.phase === 'paused'` AND `next.pauseReason === 'milestone'` | `loop-paused-milestone` |
| `prev.phase === 'paused'` AND `next.phase === 'running'` | `loop-resumed` |
| `prev.phase !== 'completed'` AND `next.phase === 'completed'` | `loop-completed` |
| `prev.phase !== 'cancelled'` AND `next.phase === 'cancelled'` | `loop-cancelled` |

`progress-updated` event is generated **inside `setProgress`** when new progress data is received.

`iteration-failed` event is generated **inside `setJudgeFeedback`** when a new entry with `approved: false` is added for an iteration that was not previously marked failed.

**Event `label` strings:**

```typescript
const ActivityLabels: Record<ActivityEventType, (e: ActivityEvent) => string> = {
  'loop-started':            (e) => `Loop started — Round ${e.iteration}`,
  'iteration-complete':      (e) => `Round ${e.iteration} complete`,
  'iteration-failed':        (e) => `Round ${e.iteration} rejected by Judge`,
  'loop-paused-hitl':        (e) => `Loop paused — awaiting review`,
  'loop-paused-milestone':   (e) => `Milestone checkpoint reached`,
  'loop-resumed':            (e) => `Loop resumed — Round ${e.iteration}`,
  'loop-completed':          (e) => `Loop completed after ${e.iteration} rounds`,
  'loop-cancelled':          (e) => `Loop cancelled at Round ${e.iteration}`,
  'progress-updated':        (e) => `Progress updated`,
};
```

---

## 6. State Machine

### 6.1 LoopPhase Transitions

```
null  (app start / no project selected)
  │
  ├─[project selected, no gulf-loop.local.md]──────────────────→ 'idle'
  │
  └─[project selected, file exists]
         │
         ├─ active=true  ───────────────────────────────────────→ 'running'
         │     │
         │     ├─[active=false, pause_reason='hitl']────────────→ 'paused'
         │     ├─[active=false, pause_reason='milestone']────────→ 'paused'
         │     │     └─[active=true again]────────────────────────→ 'running'
         │     │
         │     └─[active=false, pause_reason='']────────────────→ 'completed'
         │
         └─[file deleted (deleted=true)]──────────────────────────→ 'completed'
              (loop ended without explicit pause_reason)
```

**Invariants:**
- `'completed'` and `'cancelled'` are **terminal states**. Once entered, no further `loop-state-changed` events change the phase. Only `resetLoop()` can exit.
- `loopState = null` means no file has been seen yet for the active project. `LoopMonitor` must NOT render in this state.
- Phase never goes `'completed'` → `'running'` without `resetLoop()` first.

**Event payload priority rule (for `loop-state-changed`):**
- `{ deleted: true, content: null }` → treat as file deletion → transition to `phase = 'completed'` (S5)
- `{ deleted: false, content: null }` → file unreadable or not yet present → set `loopState = null` (S6)
- Both fields must be checked; `deleted: true` takes precedence over `content: null`.

### 6.2 IterationItem Status Derivation

```typescript
function deriveIterationStatus(
  node: IterationItem,
  loopState: LoopState,
  feedback: JudgeFeedbackEntry[]
): IterationItem['status'] {
  const isRejected = feedback.some(
    f => f.iteration === node.iteration && !f.approved
  );
  const isApproved = feedback.some(
    f => f.iteration === node.iteration && f.approved
  );

  // Final approval overrides rejection
  if (isRejected && !isApproved) return 'failed';

  if (loopState.iteration > node.iteration) return 'completed';
  if (loopState.iteration === node.iteration) return 'active';
  return 'pending';
}
```

### 6.3 ProgressSnapshot Capture

When iteration increments (`next.iteration > prev.iteration`), capture the current progress as a snapshot and attach it to the **previous** iteration's `IterationItem`:

```typescript
// Inside setLoopState, when iteration increments:
if (prev && next.iteration > prev.iteration) {
  const snapshot: ProgressSnapshot = {
    iteration: prev.iteration,
    completed: get().progress?.completed ?? [],
    decisions: get().progress?.decisions ?? [],
    uncertainties: get().progress?.uncertainties ?? [],
    remainingGap: get().progress?.remainingGap ?? [],
    confidence: get().progress?.confidence ?? null,
    nextStep: get().progress?.nextStep ?? null,
  };
  // attach snapshot to iteration prev.iteration
  get().updateIteration({ iteration: prev.iteration, progressSnapshot: snapshot });
}
```

**Final iteration snapshot (terminal state capture):**

When the loop transitions to `'completed'` or `'cancelled'`, the current iteration counter does **not** increment — so the code above never fires for the last round. Capture it explicitly on terminal transition:

```typescript
// Inside setLoopState, on terminal transition:
const isTerminal  = next.phase === 'completed' || next.phase === 'cancelled';
const wasTerminal = prev?.phase === 'completed' || prev?.phase === 'cancelled';

if (isTerminal && !wasTerminal) {
  const snapshot: ProgressSnapshot = {
    iteration: next.iteration,
    completed: get().progress?.completed ?? [],
    decisions: get().progress?.decisions ?? [],
    uncertainties: get().progress?.uncertainties ?? [],
    remainingGap: get().progress?.remainingGap ?? [],
    confidence: get().progress?.confidence ?? null,
    nextStep: get().progress?.nextStep ?? null,
  };
  get().updateIteration({ iteration: next.iteration, progressSnapshot: snapshot });
}
```

**Invariant:** every `IterationItem` that actually executed must have a `progressSnapshot`. The IterationTimeline (§7.2) and EpisodeList (§7.13.5) must never show "No details" for a completed or cancelled round.

---

## 7. Feature Specifications

### 7.1 ActivityFeed Component (NEW)

**File:** `src/components/monitor/ActivityFeed.tsx`

**Purpose:** Displays the full event stream of the loop in reverse-chronological order. Replaces the current JudgeFeedback-only "Activity" tab.

**Rendering contract:**
- Source: `activityEvents` from store.
- Display order: **newest first** (`[...events].reverse()`).
- Empty state: `"Start a loop to see activity."` — centered, `var(--text-2)` color.
- Each event row:
  - Left: colored dot (6×6px circle)
  - Center: `label` (primary text) + optional `detail` (secondary, smaller)
  - Right: relative timestamp (`formatRelativeTime(event.timestamp)`)

**Dot colors by event type:**

| Event type | Dot color |
|-----------|-----------|
| `iteration-complete` | `var(--green)` |
| `iteration-failed` | `var(--red)` |
| `loop-completed` | `var(--accent)` |
| `loop-paused-hitl` | `var(--yellow)` |
| `loop-paused-milestone` | `var(--yellow)` |
| `loop-resumed` | `var(--cyan)` |
| `loop-started` | `var(--accent)` |
| `loop-cancelled` | `var(--red)` |
| `progress-updated` | `var(--text-3)` |

**`formatRelativeTime(ts: number): string`:**
- `< 60s` → `"just now"`
- `< 60min` → `"Nm ago"` (e.g. `"3m ago"`)
- `< 24h` → `"Nh ago"`
- else → locale date string

**Full JSX:**

```tsx
export function ActivityFeed() {
  const events = useStore((s) => s.activityEvents);

  if (events.length === 0) {
    return <div className="monitor-empty">Start a loop to see activity.</div>;
  }

  return (
    <div className="activity-feed">
      {[...events].reverse().map((event) => (
        <div key={event.id} className={`activity-event activity-event--${event.type}`}>
          <span className="activity-event-dot" />
          <div className="activity-event-content">
            <span className="activity-event-label">{event.label}</span>
            {event.detail && (
              <span className="activity-event-detail">{event.detail}</span>
            )}
          </div>
          <span className="activity-event-time">
            {formatRelativeTime(event.timestamp)}
          </span>
        </div>
      ))}
    </div>
  );
}
```

**CSS:**

```css
.activity-feed {
  display: flex; flex-direction: column; gap: 1px; padding: 8px 4px;
}
.activity-event {
  display: flex; align-items: flex-start; gap: 8px;
  padding: 6px 8px; font-size: 12px;
  transition: background var(--t-fast);
}
.activity-event:hover { background: var(--bg-hover); }
.activity-event-dot {
  width: 6px; height: 6px; border-radius: 50%;
  margin-top: 4px; flex-shrink: 0; background: var(--text-3);
}
.activity-event--iteration-complete  .activity-event-dot { background: var(--green); }
.activity-event--iteration-failed    .activity-event-dot { background: var(--red); }
.activity-event--loop-completed      .activity-event-dot { background: var(--accent); }
.activity-event--loop-started        .activity-event-dot { background: var(--accent); }
.activity-event--loop-paused-hitl    .activity-event-dot { background: var(--yellow); }
.activity-event--loop-paused-milestone .activity-event-dot { background: var(--yellow); }
.activity-event--loop-resumed        .activity-event-dot { background: var(--cyan); }
.activity-event--loop-cancelled      .activity-event-dot { background: var(--red); }
.activity-event-content {
  flex: 1; display: flex; flex-direction: column; gap: 1px;
}
.activity-event-label  { color: var(--text-1); line-height: 1.4; }
.activity-event-detail { font-size: 11px; color: var(--text-2); }
.activity-event-time   { font-size: 10px; color: var(--text-3); white-space: nowrap; margin-left: 4px; }
```

---

### 7.2 IterationTimeline — Rich Snapshot Panel

**File:** `src/components/monitor/IterationTimeline.tsx` (modify existing)

**New behavior for the snapshot detail panel (right side of timeline):**

When a round node is clicked:
- `selectedIteration` state updates to the clicked iteration number.
- If `progressSnapshot` exists → render the rich panel below.
- If `progressSnapshot` is `undefined` → render: `<p className="monitor-empty">No snapshot for this round.</p>`

**Snapshot panel contract:**

```tsx
{selectedSnapshot ? (
  <div className="iteration-snapshot">
    <div className="iteration-snapshot-header">
      <span className="iteration-snapshot-title">Round {selectedIteration}</span>
      {selectedSnapshot.confidence !== null && (
        <span className="iteration-snapshot-confidence">
          {selectedSnapshot.confidence}% confidence
        </span>
      )}
    </div>

    {selectedSnapshot.completed.length > 0 && (
      <div className="snapshot-section">
        <div className="snapshot-section-label">Completed</div>
        {selectedSnapshot.completed.map((item, i) => (
          <div key={i} className="snapshot-item">{item}</div>
        ))}
      </div>
    )}

    {selectedSnapshot.nextStep && (
      <div className="snapshot-section">
        <div className="snapshot-section-label">Next goal</div>
        <div className="snapshot-item snapshot-item--accent">{selectedSnapshot.nextStep}</div>
      </div>
    )}

    {selectedSnapshot.decisions.length > 0 && (
      <div className="snapshot-section">
        <div className="snapshot-section-label">Decisions</div>
        {selectedSnapshot.decisions.map((d, i) => (
          <div key={i} className="snapshot-item">
            <strong>{d.chose}</strong>
            {d.rejected && <span className="snapshot-rejected"> vs {d.rejected}</span>}
            {d.reason && <span className="snapshot-reason"> — {d.reason}</span>}
          </div>
        ))}
      </div>
    )}

    {selectedSnapshot.uncertainties.length > 0 && (
      <div className="snapshot-section">
        <div className="snapshot-section-label">Uncertainties</div>
        {selectedSnapshot.uncertainties.map((item, i) => (
          <div key={i} className="snapshot-item snapshot-item--muted">{item}</div>
        ))}
      </div>
    )}
  </div>
) : (
  <p className="monitor-empty">No snapshot for this round.</p>
)}
```

**Node status visual spec:**

| Status | Visual |
|--------|--------|
| `completed` | Default node style |
| `active` | Accent-colored ring |
| `pending` | Dimmed / `var(--text-3)` |
| `failed` | `2px solid var(--red)` border + `rgba(248,113,113,0.1)` background |

**Sublabel rule:**
- `active` node: sublabel = `progress.nextStep` (from current progress, not snapshot)
- `nextStep` is `null` → no sublabel rendered (empty string display forbidden)

**CSS additions:**

```css
.iteration-snapshot-header {
  display: flex; justify-content: space-between; align-items: center;
  margin-bottom: 10px;
}
.iteration-snapshot-title   { font-size: 13px; font-weight: 600; color: var(--text-0); }
.iteration-snapshot-confidence { font-size: 11px; color: var(--accent); font-weight: 600; }
.snapshot-section           { margin-bottom: 8px; }
.snapshot-section-label     {
  font-size: 10px; font-weight: 700; color: var(--text-2);
  letter-spacing: 0.06em; text-transform: uppercase; margin-bottom: 4px;
}
.snapshot-item              { font-size: 12px; color: var(--text-1); padding: 2px 0; }
.snapshot-item--accent      { color: var(--accent); }
.snapshot-item--muted       { color: var(--text-2); }
.snapshot-rejected          { color: var(--red); font-size: 11px; }
.snapshot-reason            { color: var(--text-2); font-style: italic; }

.iteration-node--failed {
  border: 2px solid var(--red) !important;
  background: rgba(248, 113, 113, 0.1) !important;
}
.iteration-node--failed .iteration-node-number { color: var(--red); }
```

---

### 7.3 ProgressSummary — Expand/Collapse

**File:** `src/components/monitor/ProgressSummary.tsx` (modify existing)

**Contract:**
- If `progress === null`: return `null` (do not render the component).
- `completed.length ≤ 3`: show all items directly.
- `completed.length > 3`: show first 3 + expand button `"+N more"`.
- Expand button click → show all + button changes to `"Show less"`.
- `confidence === null`: do not render confidence display (no `0%` placeholder).

```tsx
const [showAll, setShowAll] = useState(false);
const visible = showAll ? progress.completed : progress.completed.slice(0, 3);
const extra = progress.completed.length - 3;

{progress.completed.length > 3 && (
  <button className="expand-btn" onClick={() => setShowAll(v => !v)}>
    {showAll ? 'Show less' : `+${extra} more`}
  </button>
)}
```

```css
.expand-btn {
  font-size: 11px; color: var(--accent); background: none; border: none;
  cursor: pointer; padding: 2px 0; transition: opacity var(--t-fast);
}
.expand-btn:hover { opacity: 0.7; }
```

---

### 7.4 JudgeFeedback — Formatting Improvements

**File:** `src/components/monitor/JudgeFeedback.tsx` (modify existing)

**Contract:**
- Empty state (`judgeFeedback.length === 0`): `"No judge feedback yet."` (centered).
- Timestamp: format with `formatFeedbackTime`. ISO parse failure → show raw string.
- Badge: show `approved` status + iteration number if `entry.iteration !== null`.
- "View raw" toggle: shows `entry.rawText` in a `<pre>` block.

```typescript
function formatFeedbackTime(ts: string): string {
  if (!ts) return '';
  try {
    const d = new Date(ts);
    if (isNaN(d.getTime())) return ts;
    return d.toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit' });
  } catch { return ts; }
}
```

---

### 7.5 SessionReport — Completed Loop View

**File:** `src/components/monitor/LoopMonitor.tsx` (modify existing completed branch)

**Trigger:** `loopState.phase === 'completed' || loopState.phase === 'cancelled'`

**Layout (in order):**

1. **GoalBanner** — `progress.originalGoal ?? "No goal recorded"` (never empty div)

2. **Stats Row** — horizontal flex, equal-width cards:
   - Always: `loopState.iteration` / `"Rounds"`, `progress.completed.length ?? 0` / `"Tasks done"`, `progress.confidence ?? '—'` / `"Confidence"`
   - Judge mode only (`judgeEnabled: true`): `loopState.consecutiveRejections` / `"Rejections"`

3. **Completed tasks** — full list, collapse/expand at 3. Left border `2px solid var(--green)`.

4. **Remaining** — `progress.remainingGap` items. **Only render when `phase === 'cancelled'` AND `progress.remainingGap.length > 0`**. Left border `2px solid var(--yellow)`.

5. **Decisions** — `progress.decisions` full list.

6. **Uncertainties** — `progress.uncertainties` full list.

7. **Per-round log** — `iterations` array rendered as `IterationReportRow` components (expand/collapse each).

8. **"New Goal" button** — calls `resetLoop()` + triggers `setShowWizard(true)`.

**IterationReportRow contract:**
- Default: collapsed (ChevronRight icon).
- Expand: shows `progressSnapshot` fields + `gitCommits` list.
- Git diff: on-demand via `invoke('get_git_diff', { projectPath, commitHash })`. Show loading state while fetching.
- Diff text in `<pre>` with horizontal scroll.

**Stats CSS:**

```css
.result-summary-row {
  display: flex; gap: 16px; margin-bottom: 16px;
  padding-bottom: 16px; border-bottom: 1px solid var(--border);
}
.result-stat          { text-align: center; flex: 1; }
.result-stat-value    { font-size: 22px; font-weight: 700; color: var(--accent); font-family: var(--font-mono); }
.result-stat-label    { font-size: 10px; color: var(--text-2); margin-top: 2px; letter-spacing: 0.04em; }
```

---

### 7.6 Completed Project History View

**File:** `src/components/projects/ProjectList.tsx` (modify `handleProjectClick`)

**Contract:**

When a project with `status === 'completed' || status === 'cancelled'` is clicked:

1. Call `invoke('load_session', { projectPath: project.path })`.
2. If session data exists:
   a. Parse the JSON.
   b. Call `setIterations(data.iterations ?? [])`.
   c. Call `setProgress(data.progress ?? null, null)`.
   d. Call `setJudgeFeedback(data.judgeFeedback ?? [])`.
   e. Build a **synthetic `LoopState`** and call `setLoopState(syntheticState)`.
3. If no session data: show `QuickRestartView` (start fresh).

**Synthetic LoopState construction:**

```typescript
const maxIter = data.iterations?.length > 0
  ? Math.max(...data.iterations.map((i: IterationItem) => i.iteration))
  : 0;

const syntheticState: LoopState = {
  active: false,
  phase: project.status as 'completed' | 'cancelled',
  iteration: maxIter,
  maxIterations: 50,
  completionPromise: 'COMPLETE',
  judgeEnabled: data.iterations?.some((i: IterationItem) =>
    i.status === 'failed') ?? false,
  consecutiveRejections: 0,
  hitlThreshold: 5,
  loopMode: 'basic',
  autonomous: false,
  branch: null,
  baseBranch: null,
  milestoneEvery: 0,
  worktreePath: '',
  structuredMemory: false,
  forceMax: false,
  pauseReason: '',
  startedAt: null,
  originalPrompt: data.progress?.originalGoal ?? null,
};
```

---

### 7.7 JudgeEvolution Display

**File:** `src/components/memory/MemoryBrowser.tsx` (add `JudgeEvolutionLog` section at bottom)

**Contract:**
- Source: `judgeEvolution` from store.
- If `judgeEvolution === null`: do not render section (no empty placeholder).
- Parse lines with regex; skip malformed lines silently.
- Display newest first.
- `approved` entries: green dot `✓`.
- `rejected` entries: red dot `✗`.

**Watcher is already in place.** Ensure App.tsx listener calls `setJudgeEvolution`:

```typescript
listen<{ content: string | null }>('judge-evolution-changed', (event) => {
  useStore.getState().setJudgeEvolution(event.payload.content ?? null);
});
```

---

### 7.8 AskUserQuestion GUI Overlay

**Files:** `src/components/monitor/QuestionPromptOverlay.tsx` (NEW), `src-tauri/src/process/mod.rs` (modify)

**Purpose:** Detect `AskUserQuestion` prompts in PTY output and render clickable choice buttons instead of requiring terminal interaction.

**PTY pattern to detect:**

```
? [question text]
  1. [choice 1]
  2. [choice 2]
  ...

Your choice (1-N):
```

**Rust implementation (`process/mod.rs`):**

```rust
#[derive(Serialize, Clone)]
struct QuestionBlock {
    question: String,
    choices: Vec<String>,   // stripped of "N. " prefix
}

fn detect_question_block(buffer: &str) -> Option<QuestionBlock> {
    // 1. Find line starting with "? "
    // 2. Collect "  N. ..." lines after it
    // 3. Detect "Your choice" line → trigger emit
    // Emit: app.emit("claude-question-prompt", &block)
}
```

- Buffer per-project. Accumulate lines until "Your choice" is detected, then emit and clear buffer.
- Maximum 10 choices parsed.

**Store additions (`settingsSlice.ts`):**

```typescript
activeQuestion: { question: string; choices: string[] } | null;
setActiveQuestion: (q: { question: string; choices: string[] } | null) => void;
```

**App.tsx listener:**

```typescript
listen<{ question: string; choices: string[] }>('claude-question-prompt', (event) => {
  // Guard: only show overlay when a loop is active
  if (!useStore.getState().loopState) return;
  useStore.getState().setActiveQuestion(event.payload);
});
```

**Component contract:**

```tsx
// src/components/monitor/QuestionPromptOverlay.tsx
export function QuestionPromptOverlay() {
  const activeQuestion = useStore((s) => s.activeQuestion);
  if (!activeQuestion) return null;

  const handleSelect = async (idx: number) => {
    const projectId = useStore.getState().activeProjectId ?? '';
    await invoke('send_command', { projectId, data: `${idx + 1}\r` });
    setTimeout(() => useStore.getState().setActiveQuestion(null), 200);
  };

  return (
    <div className="question-overlay">
      <div className="question-box">
        <div className="question-text">{activeQuestion.question}</div>
        <div className="question-choices">
          {activeQuestion.choices.slice(0, 10).map((choice, idx) => (
            <button
              key={idx}
              className="question-choice-btn"
              onClick={() => handleSelect(idx)}
              type="button"
            >
              <span className="question-choice-num">{idx + 1}</span>
              <span className="question-choice-text">{choice}</span>
            </button>
          ))}
        </div>
        <button
          className="question-dismiss"
          onClick={() => useStore.getState().setActiveQuestion(null)}
          type="button"
        >
          Type manually
        </button>
      </div>
    </div>
  );
}
```

**Render location:** Inside `LoopMonitor.tsx` as an absolute overlay (z-index: 200).

**CSS:**

```css
.question-overlay {
  position: fixed; inset: 0; background: rgba(0,0,0,0.6);
  display: flex; align-items: center; justify-content: center;
  z-index: 200; backdrop-filter: blur(4px);
}
.question-box {
  background: var(--bg-1); border: 1px solid var(--border);
  padding: 20px; max-width: 480px; width: 90%;
  display: flex; flex-direction: column; gap: 12px;
}
.question-text { font-size: 14px; color: var(--text-0); font-weight: 600; line-height: 1.5; }
.question-choices { display: flex; flex-direction: column; gap: 6px; }
.question-choice-btn {
  display: flex; align-items: center; gap: 10px; padding: 8px 12px;
  background: var(--bg-2); border: 1px solid var(--border); cursor: pointer;
  color: var(--text-1); font-size: 13px; text-align: left;
  transition: border-color var(--t-fast), background var(--t-fast);
}
.question-choice-btn:hover { border-color: var(--accent); background: var(--accent-glow); }
.question-choice-num {
  min-width: 20px; height: 20px; background: var(--bg-3); color: var(--accent);
  font-size: 11px; font-weight: 700; display: flex; align-items: center;
  justify-content: center; font-family: var(--font-mono);
}
.question-dismiss {
  font-size: 11px; color: var(--text-2); background: none; border: none;
  cursor: pointer; align-self: flex-end; padding: 4px 8px;
  transition: color var(--t-fast);
}
.question-dismiss:hover { color: var(--text-1); }
```

---

### 7.9 Raw Log Viewer (Developer Mode)

**File:** `src/components/layout/RightPanel.tsx` (add raw log tab when `advancedMode: true`)

**Contract:**
- Only rendered when `advancedMode === true`. Not in DOM otherwise.
- `rawProgressContent === null` → show `"No data"` in that section (not a blank `<pre>`).
- `loopState === null` → show `"No state"` for the loop state section.

```tsx
{advancedMode && rightTab === 'raw' && (
  <div className="raw-log-view">
    <div className="raw-log-section">
      <div className="raw-log-title">gulf-loop state</div>
      <pre className="raw-log-content">
        {loopState ? JSON.stringify(loopState, null, 2) : 'No state'}
      </pre>
    </div>
    <div className="raw-log-section">
      <div className="raw-log-title">progress.txt</div>
      <pre className="raw-log-content">{rawProgressContent ?? 'No data'}</pre>
    </div>
  </div>
)}
```

**Add "raw" to TABS in RightPanel when `advancedMode: true`:**

```typescript
const TABS_BASE = [
  { id: 'activity', icon: <Zap size={15} />,         label: 'Activity' },
  { id: 'memory',   icon: <BrainCircuit size={15} />, label: 'Memory' },
  { id: 'settings', icon: <Settings2 size={15} />,    label: 'Settings' },
] as const;

const TABS_DEV = [
  ...TABS_BASE,
  { id: 'raw', icon: <Code2 size={15} />, label: 'Raw' },
] as const;

const TABS = advancedMode ? TABS_DEV : TABS_BASE;
```

---

### 7.10 Max Iterations Default & Anti-Early-Completion

**File:** `src/lib/rubric-generator.ts` (modify `buildGulfLoopPrompt`)

**Default values:**
- `maxIterations`: `50` (was `30`)
- `milestoneEvery`: `0` (disabled by default)

**Required instructions appended to every gulf-loop prompt:**

```
---
LOOP EXECUTION RULES (do not override):
- Use ALL available iterations to build incrementally.
- Do NOT output the completion signal in iteration 1 unless every requirement is demonstrably met and tested.
- Each iteration must deliver a meaningful, atomic, runnable increment.
- Update progress.txt at the end of EVERY iteration using the exact format specified.
```

**Bug fix — multi-line prompt must be flattened before PTY transmission:**

`buildGulfLoopPrompt` produces a multi-line string with many `\n` characters. When this string is sent raw to the PTY, each `\n` is interpreted as Enter by the shell — only the first line is submitted as a command; the rest become separate inputs.

**Rule:** before passing the prompt to `send_command_enter`, replace every `\r?\n` with a single space.

```typescript
// rubric-generator.ts (or call-site in QuickRestartView / GoalWizard)
const singleLinePrompt = buildGulfLoopPrompt(goal, rubric)
  .replace(/\r?\n/g, ' ')
  .replace(/\s{2,}/g, ' ')   // collapse multiple spaces
  .trim();

await invoke('send_command_enter', { projectId, data: `/gulf-loop:start "${singleLinePrompt}"` });
```

**Contract:**
- The final string sent to the PTY must contain **zero `\n` or `\r` characters**.
- Escape any `"` characters inside the prompt before embedding in the shell command string.

---

### 7.11 PTY Command Timing & Gate Fixes

#### 7.11.1 Command Timing — Streaming-Ready Detection + 2-Phase Send

**Root cause (confirmed):** Two independent bugs:
1. **Fixed 8s sleep is unreliable** — MCP server loading can take longer; Claude may not yet be at the `>` prompt when the command arrives → command ignored.
2. **Atomic `command+\r` write breaks ink TUI** — Claude Code (ink/React TUI) processes PTY input as a "large paste". Sending `command\r` as a single `write_all` means `\r` arrives while the TUI is still processing the paste buffer → Enter is not registered and the command is never submitted.

**Solution:** Two coordinated changes — stream-based readiness detection in `process/mod.rs`, and 2-phase send in `spawn_and_send`.

---

**Change A — `ClaudeProcess` ready flag (`src-tauri/src/process/mod.rs`):**

```rust
use std::sync::{Arc, atomic::{AtomicBool, Ordering}};

pub struct ClaudeProcess {
    // existing fields ...
    pub ready: Arc<AtomicBool>,   // NEW: true once Claude prompt is detected
}
```

The PTY reader thread already emits `claude-output` chunks. Extend it to also call `is_claude_prompt()` on each chunk:

```rust
// Inside the reader thread, after emitting claude-output:
let clean = strip_ansi(&chunk_str);
if !session.ready.load(Ordering::Relaxed) && is_claude_prompt(&clean) {
    session.ready.store(true, Ordering::Relaxed);
}
```

**Helper functions:**

```rust
/// Strip ANSI escape sequences (ESC [ ... m / ESC [ ... H etc.)
fn strip_ansi(s: &str) -> String {
    // regex: \x1b\[[0-9;]*[A-Za-z]
    // or use the `strip-ansi-escapes` crate
}

/// Detect Claude Code's interactive prompt
fn is_claude_prompt(s: &str) -> bool {
    s.contains("> ") || s.contains("❯ ") || s.contains("\n>")
}

/// Public API for spawn_and_send to poll
pub fn is_session_ready(project_id: &str) -> bool {
    let sessions = SESSIONS.lock().unwrap();
    sessions.get(project_id).map(|s| s.ready.load(Ordering::Relaxed)).unwrap_or(false)
}
```

**Reset:** `ready` is reset to `false` when a new session is spawned (in `spawn_claude_inner`).

---

**Change B — Streaming wait + 2-phase send (`src-tauri/src/lib.rs`):**

```rust
#[tauri::command]
async fn spawn_and_send(
    app: AppHandle,
    project_id: String,
    project_path: String,
    command: String,   // already-flattened single-line string
    cols: u16,
    rows: u16,
) -> Result<(), String> {
    // 1. Spawn Claude PTY session
    spawn_claude_inner(&app, &project_id, &project_path, cols, rows).await?;

    // 2. Stream-based readiness wait
    //    - Minimum 3s (prevents false positives from early partial output)
    //    - 300ms polling interval
    //    - +300ms buffer after ready signal (TUI fully settled)
    //    - 15s hard timeout fallback
    let start = std::time::Instant::now();
    let min_wait   = std::time::Duration::from_secs(3);
    let buffer     = std::time::Duration::from_millis(300);
    let hard_limit = std::time::Duration::from_secs(15);
    let poll       = std::time::Duration::from_millis(300);

    loop {
        tokio::time::sleep(poll).await;
        let elapsed = start.elapsed();

        if elapsed >= hard_limit {
            break;  // proceed anyway (best-effort fallback)
        }
        if elapsed >= min_wait && is_session_ready(&project_id) {
            tokio::time::sleep(buffer).await;  // settle buffer
            break;
        }
    }

    // 3. Verify session is still alive (auth failure, early exit)
    if !is_session_active_inner(&project_id) {
        return Err(format!("Session {} exited before command could be sent", project_id));
    }

    // 4. Phase 1: write command text (PTY input buffer — treated as paste)
    let mut sessions = SESSIONS.lock().map_err(|e| e.to_string())?;
    let session = sessions.get_mut(&project_id)
        .ok_or_else(|| format!("No session for {}", project_id))?;
    session.writer.write_all(command.as_bytes())
        .map_err(|e| e.to_string())?;
    drop(sessions);  // release lock before sleep

    // 5. Phase 2: wait for TUI to process paste, then send Enter
    tokio::time::sleep(std::time::Duration::from_millis(500)).await;

    let mut sessions = SESSIONS.lock().map_err(|e| e.to_string())?;
    let session = sessions.get_mut(&project_id)
        .ok_or_else(|| format!("No session for {}", project_id))?;
    session.writer.write_all(b"\r")
        .map_err(|e| e.to_string())?;

    Ok(())
}
```

**Why 2-phase send:**
- Claude Code (ink) processes PTY input as keyboard events. When `command\r` arrives as one chunk, the ink event loop receives characters and Enter simultaneously. Large commands are processed as "paste" — the `\r` can be consumed by the paste handler before the command text is fully inserted into the input buffer.
- Sending command text first → 500ms pause → `\r` separately ensures the ink input field has fully processed all characters before Enter is delivered.

**Why `tokio::time::sleep` not `std::thread::sleep`:**
- `std::thread::sleep` blocks the Tauri worker thread entirely. PTY reader threads, file watch events, and other Tauri commands cannot proceed during the wait.
- `tokio::time::sleep` yields the executor — PTY output continues streaming (needed for readiness detection), file watch events fire normally, and other commands remain responsive.

**Frontend call site (QuickRestartView, GoalWizard) — unchanged:**

```typescript
const cmd = buildFlatCommand(prompt);  // flatten \n → space
await invoke('spawn_and_send', {
  projectId,
  projectPath,
  command: cmd,
  cols: 220,
  rows: 50,
});
```

**Invariants:**
- The entire wait + 2-phase send happens in the Rust async runtime. The frontend `invoke` returns only after `\r` has been written.
- No frontend `setTimeout`, `clearTimeout`, or `listen('claude-output')` needed for loop start.
- `spawn_claude` (standalone, no command) remains available for terminal-only mode.
- Component unmount during the wait does not cause a double-send — the backend completes independently of the frontend lifecycle.

---

#### 7.11.2 MilestoneGate — Missing `projectId` Fix

**Problem:** `invoke('send_command', { data: '...' })` omits `projectId`, causing the Tauri command to fail silently (writes to no PTY).

**Required fix:** All PTY writes in `MilestoneGate.tsx` must use `send_command_enter` with explicit `projectId`.

```typescript
// MilestoneGate.tsx — every invoke call must follow this pattern:
const projectId = useStore.getState().activeProjectId ?? '';

// Resume loop
await invoke('send_command_enter', { projectId, data: '/gulf-loop:resume' });

// Cancel loop
await invoke('send_command_enter', { projectId, data: '/gulf-loop:cancel' });
```

**Contract:**
- `projectId` must be read from the store at call time (not captured at component mount time, to avoid stale closures).
- Use `send_command_enter` (not `send_command`) for all gulf-loop slash commands so the shell receives a proper newline.

---

#### 7.11.3 HITLGate / MilestoneGate — Unified `send_command_enter`

**Problem:** Resume and cancel calls are inconsistent across `HITLGate` and `MilestoneGate` — some use `send_command`, some use `send_command_enter`, some omit `projectId`.

**Rule:** Every resume/cancel PTY write in both gates must use the same pattern:

```typescript
await invoke('send_command_enter', {
  projectId: useStore.getState().activeProjectId ?? '',
  data: '<slash-command>',
});
```

**Exhaustive list of calls to fix:**

| Component | Action | Correct call |
|-----------|--------|-------------|
| `MilestoneGate` | Resume | `invoke('send_command_enter', { projectId, data: '/gulf-loop:resume' })` |
| `MilestoneGate` | Cancel | `invoke('send_command_enter', { projectId, data: '/gulf-loop:cancel' })` |
| `HITLGate` | Resume | `invoke('send_command_enter', { projectId, data: '/gulf-loop:resume' })` |
| `HITLGate` | Cancel | `invoke('send_command_enter', { projectId, data: '/gulf-loop:cancel' })` |

No `send_command` (without `_enter`) should be used for slash commands. `send_command` is reserved for raw PTY data that does not need a trailing newline (e.g., keystroke injection in `QuestionPromptOverlay`).

---

#### 7.11.4 `setActiveProject` Must Be Called Before `spawn_and_send`

**Root cause:**

`App.tsx` event handlers gate every store update on `activeProjectId`:

```typescript
// App.tsx — loop-state-changed handler (and all other file-watch handlers)
const { activeProjectId, updateProject } = useStore.getState();
if (activeProjectId) {          // ← null → entire event silently dropped
  updateProject(activeProjectId, { status: 'running' });
}
```

`spawn_and_send` waits 8 seconds in the Rust backend before returning. During those 8 seconds:
- `start_watcher` is already running and emitting events
- The gulf-loop state file can be created and trigger `loop-state-changed`
- If `activeProjectId` is still `null` at that point, all events are silently dropped and `LoopMonitor` never appears

**Required fix:** call `setActiveProject(projectId, projectPath)` **synchronously before** `invoke('spawn_and_send', ...)`.

```typescript
// QuickRestartView.tsx / GoalWizard.tsx — correct ordering
const startLoop = async () => {
  // 1. Set active project FIRST — must happen before any await
  setActiveProject(project.id, project.path);
  await invoke('start_watcher', { projectPath: project.path });

  // 2. Now spawn + send — watcher events during the 8s wait will route correctly
  const cmd = buildFlatCommand(prompt);
  await invoke('spawn_and_send', {
    projectId: project.id,
    projectPath: project.path,
    command: cmd,
    cols: 220,
    rows: 50,
  });
};
```

**Invariant:** `activeProjectId` in the store must be non-null before the first `start_watcher` call for that project. Any ordering that allows watcher events to fire while `activeProjectId === null` is incorrect.

**Unmount safety:** The `invoke('spawn_and_send', ...)` call resolves after ~8 seconds. The calling component (`QuickRestartView` / `GoalWizard`) may unmount before that (e.g., the loop-state-changed event causes a route change to `LoopMonitor`). The resolved Promise value must not trigger a `setState` call that is conditional on mount state. Since the function only calls `invoke` and the result is not used to set local component state, this is inherently safe — but any future chained `.then()` that touches local state must be guarded with an `isMounted` ref.

---

#### 7.11.5 Enter Key Not Delivered — Confirmed Root Cause & Fix

**Root cause (confirmed):** Claude Code uses ink (React TUI). When the PTY receives `command\r` as a single `write_all`, the ink event loop processes this as a "large paste" operation. During paste processing, `\r` arrives while the command text is still being inserted into the input buffer — ink's paste handler consumes the carriage return as part of the paste, and the command is never submitted.

**Fix:** The 2-phase send in `spawn_and_send` (§7.11.1) resolves this:
- Phase 1: `write_all(command.as_bytes())` — command text only, no `\r`
- 500ms sleep — ink TUI processes all pasted characters
- Phase 2: `write_all(b"\r")` — Enter delivered after paste is fully processed

**`send_command_enter` contract (for gate commands — resume/cancel):**

Gate commands (MilestoneGate, HITLGate) are short slash commands (`/gulf-loop:resume`, `/gulf-loop:cancel`). These are not affected by the paste-split issue (they are short enough for the ink event loop to handle atomically). For these, `send_command_enter` may continue to write `"{data}\r"` as a single `write_all`:

```rust
async fn send_command_enter_inner(project_id: &str, data: &str) -> Result<(), String> {
    let mut sessions = SESSIONS.lock().map_err(|e| e.to_string())?;
    let session = sessions.get_mut(project_id)
        .ok_or_else(|| format!("No session for {}", project_id))?;

    // Short commands: single write is safe (no paste-split needed)
    let payload = format!("{}\r", data);
    session.writer.write_all(payload.as_bytes())
        .map_err(|e| e.to_string())?;

    Ok(())
}
```

**Rule summary:**
- `spawn_and_send` (long prompt): 2-phase send — text first, `\r` after 500ms
- `send_command_enter` (short slash commands): single `write_all("{data}\r")` — `\r` only, no `\n`
- Never use `\r\n` — PTY line discipline handles line ending from `\r` alone

---

### 7.12 Usage Tracking

#### 7.12.1 Current State & Gaps

| Item | Status | Gap |
|------|--------|-----|
| `loopState.iteration / maxIterations` display | ✅ Working | — |
| Estimated cost KRW (`estimateCostKrw(iteration)`) | ✅ Working | Static — only re-renders on other state changes |
| Elapsed time (`formatElapsed(startedAt)`) | ⚠️ Partial | No live timer; only re-renders on state changes |
| `loopStartTimeMs` field in store | ❌ Dead store | Set once, never read; all UI uses `loopState.startedAt` |
| `formatElapsedMinutes()` in cost-calculator.ts | ❌ Dead code | Never imported or called |
| SessionReport stats row | ❌ Missing | Elapsed time and cost not shown in completed view |

#### 7.12.2 Time Source — Canonical Rule

**There must be exactly one source of truth for loop start time.**

Use `loopState.startedAt` (ISO string from the gulf-loop state file) as the canonical source:
- It reflects the actual time the loop agent wrote the file, not the time the UI processed the event.
- It survives session restore (`load_session` populates `loopState.startedAt` from the saved state).

**`loopStartTimeMs` role:** keep the field for session persistence (it is already serialised into `session.json`). Do not display it directly; do not remove it. Consider it an internal bookmark, not a display source.

**Remove `formatElapsedMinutes`** from `cost-calculator.ts` — it duplicates `formatElapsed` in `explainer.ts` and is never called.

#### 7.12.3 Live Elapsed Time — Interval Re-render

`TokenUsageCard` and `StatusBar` only re-render when Zustand state changes. If the loop is idle between iterations, elapsed time freezes on screen.

**Fix:** add a 30-second interval inside `TokenUsageCard` (and `StatusBar` if it shows elapsed time) that forces a local state tick to trigger re-render.

```typescript
// TokenUsageCard.tsx
const [, forceRender] = useReducer(n => n + 1, 0);

useEffect(() => {
  if (loopState?.phase !== 'running') return;
  const id = setInterval(forceRender, 30_000);
  return () => clearInterval(id);
}, [loopState?.phase]);
```

**Contract:**
- The interval runs **only while `phase === 'running'`**. It must not run during `paused`, `completed`, `cancelled`, or `idle`.
- The interval is cleared on unmount and on phase change.
- 30s granularity is sufficient — elapsed display resolves to minutes.

#### 7.12.4 Cost Estimation Constants

```typescript
// src/lib/cost-calculator.ts — authoritative constants
export const TOKENS_PER_ITERATION = 8_000;      // conservative per-iteration estimate
export const COST_PER_MILLION_KRW = 4_800;       // ~$3.50/M × 1,380 KRW/USD

export function estimateCostKrw(iterations: number): number {
  return Math.round((iterations * TOKENS_PER_ITERATION / 1_000_000) * COST_PER_MILLION_KRW);
}

export function formatCostKrw(krw: number): string {
  return `₩${krw.toLocaleString('ko-KR')}`;
}
// formatElapsedMinutes — DELETE. Use formatElapsed() from explainer.ts instead.
```

#### 7.12.5 TokenUsageCard Display Contract

**Required display fields (all must be present when `loopState !== null`):**

| Field | Source | Format |
|-------|--------|--------|
| Round progress | `loopState.iteration / loopState.maxIterations` | `"N / M 라운드"` + progress bar |
| Estimated cost | `estimateCostKrw(loopState.iteration)` | `"약 ₩N,NNN (추정)"` |
| Elapsed time | `formatElapsed(loopState.startedAt)` | `"방금"` \| `"Nm 경과"` \| `"Nh 경과"` |

**Empty / null guards:**
- `loopState === null` → render nothing (no placeholder card).
- `loopState.startedAt === null` → omit elapsed time row entirely (no "0분 경과" placeholder).
- `loopState.iteration === 0` → show `"₩0 (추정)"`.

#### 7.12.6 SessionReport — Cost & Time in Stats Row

The completed view stats row (§7.5) must include two additional stat cards:

| Stat | Source | Format | Condition |
|------|--------|--------|-----------|
| Elapsed time | `formatElapsed(loopState.startedAt)` | `"Nh Nm"` | `startedAt !== null` |
| Estimated cost | `formatCostKrw(estimateCostKrw(loopState.iteration))` | `"₩N,NNN"` | always |

Updated stats row (replacing the §7.5 spec):

```tsx
<div className="result-summary-row">
  <div className="result-stat">
    <div className="result-stat-value">{loopState.iteration}</div>
    <div className="result-stat-label">Rounds</div>
  </div>
  <div className="result-stat">
    <div className="result-stat-value">{progress?.completed.length ?? 0}</div>
    <div className="result-stat-label">Tasks done</div>
  </div>
  <div className="result-stat">
    <div className="result-stat-value">{progress?.confidence ?? '—'}</div>
    <div className="result-stat-label">Confidence</div>
  </div>
  {loopState.startedAt && (
    <div className="result-stat">
      <div className="result-stat-value">{formatElapsed(loopState.startedAt)}</div>
      <div className="result-stat-label">Elapsed</div>
    </div>
  )}
  <div className="result-stat">
    <div className="result-stat-value">
      {formatCostKrw(estimateCostKrw(loopState.iteration))}
    </div>
    <div className="result-stat-label">Est. cost</div>
  </div>
  {loopState.judgeEnabled && (
    <div className="result-stat">
      <div className="result-stat-value">{loopState.consecutiveRejections}</div>
      <div className="result-stat-label">Rejections</div>
    </div>
  )}
</div>
```

---

### 7.13 Narrative Memory Panel — Non-Developer View

**Problem with current MemoryBrowser:** It shows raw lists copied directly from `progress.txt` — technical jargon, bullet fragments, no context. A non-developer cannot understand what the AI actually decided, why, or what it means for the project.

**Goal:** Replace `MemoryBrowser` with a narrative-first panel that tells the story of the project. Every section must be readable by someone with no coding background.

**File:** `src/components/memory/MemoryBrowser.tsx` (full rewrite)
**File:** `src/lib/explainer.ts` (add narrative sentence functions)

---

#### 7.13.1 Panel Layout (top to bottom)

```
┌──────────────────────────────────────┐
│  PROJECT GOAL                        │  GoalCard
│  [originalGoal plain text]           │
├──────────────────────────────────────┤
│  WHERE WE ARE                        │  ProgressCard
│  Round 4 · 72% confidence           │
│  [ConfidenceBar + contextual label]  │
├──────────────────────────────────────┤
│  WHAT'S BEING WORKED ON NOW          │  (only when phase=running/paused)
│  → [nextStep]                        │  FocusCard
├──────────────────────────────────────┤
│  STORY SO FAR                        │  EpisodeList
│  ▾ Round 3 — Passed ✓               │
│    What was built: ...               │
│    Key decision: ...                 │
│  ▾ Round 2 — Rejected ✗             │
│    ...                               │
├──────────────────────────────────────┤
│  CHOICES & WHY                       │  DecisionNarrativeList
│  The AI chose TypeScript over        │
│  JavaScript because ...              │
├──────────────────────────────────────┤
│  OPEN QUESTIONS                      │  (only when uncertainties.length > 0)
│  The AI flagged 2 open questions:    │  UncertaintyList
│  • [item]                            │
├──────────────────────────────────────┤
│  QUALITY CHECK HISTORY               │  (only when judgeFeedback.length > 0)
│  Round 3: ✗ Found an issue           │  JudgeNarrativeList
│  Round 5: ✓ Passed                   │
└──────────────────────────────────────┘
```

---

#### 7.13.2 GoalCard

**Source:** `progress.originalGoal`

```tsx
{progress?.originalGoal && (
  <section className="narrative-section">
    <div className="narrative-section-label">Project Goal</div>
    <div className="narrative-goal-text">{progress.originalGoal}</div>
  </section>
)}
```

- If `originalGoal === null`: section not rendered (no placeholder).

---

#### 7.13.3 ProgressCard + ConfidenceBar

**Source:** `loopState.iteration`, `loopState.maxIterations`, `progress.confidence`

```tsx
<section className="narrative-section">
  <div className="narrative-section-label">Where We Are</div>
  <div className="narrative-progress-row">
    <span className="narrative-round-count">
      Round {loopState.iteration} of {loopState.maxIterations}
    </span>
    {progress?.confidence !== null && (
      <ConfidenceBar confidence={progress.confidence} />
    )}
  </div>
</section>
```

**ConfidenceBar contract:**

```tsx
function ConfidenceBar({ confidence }: { confidence: number }) {
  return (
    <div className="confidence-bar-wrap">
      <div className="confidence-bar-track">
        <div
          className="confidence-bar-fill"
          style={{ width: `${confidence}%` }}
        />
      </div>
      <div className="confidence-bar-label">
        <span className="confidence-pct">{confidence}%</span>
        <span className="confidence-text">{confidenceLabel(confidence)}</span>
      </div>
    </div>
  );
}
```

**`confidenceLabel(n: number): string`:**

| Range | Label |
|-------|-------|
| 30–45 | `"Early stages — foundations being laid"` |
| 46–60 | `"Making progress — core features taking shape"` |
| 61–75 | `"More than halfway — major work completed"` |
| 76–89 | `"Nearly complete — polishing and testing"` |
| 90–100 | `"Complete — final verification underway"` |

**CSS:**

```css
.confidence-bar-wrap   { display: flex; flex-direction: column; gap: 4px; flex: 1; }
.confidence-bar-track  { height: 4px; background: var(--bg-3); border-radius: 2px; overflow: hidden; }
.confidence-bar-fill   { height: 100%; background: var(--accent); transition: width 0.4s ease; }
.confidence-bar-label  { display: flex; gap: 8px; align-items: baseline; }
.confidence-pct        { font-size: 13px; font-weight: 700; color: var(--accent); font-family: var(--font-mono); }
.confidence-text       { font-size: 11px; color: var(--text-2); }
```

---

#### 7.13.4 FocusCard — What the AI Is Working on Now

**Source:** `progress.nextStep`
**Condition:** render only when `loopState?.phase === 'running' || loopState?.phase === 'paused'`

```tsx
{progress?.nextStep && (loopState?.phase === 'running' || loopState?.phase === 'paused') && (
  <section className="narrative-section narrative-section--accent">
    <div className="narrative-section-label">Working on now</div>
    <div className="narrative-focus-text">
      <span className="narrative-focus-arrow">→</span>
      {progress.nextStep}
    </div>
  </section>
)}
```

```css
.narrative-section--accent { border-left: 2px solid var(--accent); padding-left: 10px; }
.narrative-focus-text      { font-size: 13px; color: var(--text-0); display: flex; gap: 8px; align-items: flex-start; }
.narrative-focus-arrow     { color: var(--accent); font-weight: 700; flex-shrink: 0; }
```

---

#### 7.13.5 EpisodeList — Story So Far

**Source:** `iterations[]` (from store), each with optional `progressSnapshot` and `gitCommits`

Each `IterationItem` is one episode. Render newest-first.

**Episode row contract:**

```tsx
function EpisodeRow({ item, feedback }: { item: IterationItem; feedback: JudgeFeedbackEntry[] }) {
  const [expanded, setExpanded] = useState(false);
  const roundFeedback = feedback.filter(f => f.iteration === item.iteration);
  const wasRejected = roundFeedback.some(f => !f.approved);
  const wasPassed   = roundFeedback.some(f =>  f.approved);

  return (
    <div className={`episode-row episode-row--${item.status}`}>
      <button className="episode-header" onClick={() => setExpanded(v => !v)}>
        <span className="episode-chevron">{expanded ? '▾' : '▸'}</span>
        <span className="episode-title">Round {item.iteration}</span>
        <span className={`episode-badge episode-badge--${wasPassed ? 'passed' : wasRejected ? 'failed' : item.status}`}>
          {wasPassed ? '✓ Passed' : wasRejected ? '✗ Rejected' : item.status === 'active' ? '⟳ Running' : '—'}
        </span>
      </button>

      {expanded && item.progressSnapshot && (
        <div className="episode-body">
          {item.progressSnapshot.completed.length > 0 && (
            <div className="episode-field">
              <div className="episode-field-label">What was built</div>
              {item.progressSnapshot.completed.map((c, i) => (
                <div key={i} className="episode-field-item">{c}</div>
              ))}
            </div>
          )}

          {item.progressSnapshot.decisions.length > 0 && (
            <div className="episode-field">
              <div className="episode-field-label">Key decision</div>
              {item.progressSnapshot.decisions.slice(0, 1).map((d, i) => (
                <div key={i} className="episode-field-item episode-field-item--decision">
                  {decisionToSentence(d)}
                </div>
              ))}
            </div>
          )}

          {roundFeedback.filter(f => !f.approved).map((fb, i) => (
            <div key={i} className="episode-field episode-field--rejected">
              <div className="episode-field-label">Quality check issue</div>
              <div className="episode-field-item">{fb.summary || 'No details recorded.'}</div>
            </div>
          ))}
        </div>
      )}

      {expanded && !item.progressSnapshot && (
        <div className="episode-body episode-body--empty">No details recorded for this round.</div>
      )}
    </div>
  );
}
```

```css
.episode-row         { border-bottom: 1px solid var(--border); }
.episode-header      { width: 100%; display: flex; align-items: center; gap: 8px; padding: 8px 12px;
                       background: none; border: none; cursor: pointer; text-align: left; }
.episode-header:hover { background: var(--bg-hover); }
.episode-chevron     { font-size: 10px; color: var(--text-3); width: 10px; flex-shrink: 0; }
.episode-title       { font-size: 12px; color: var(--text-1); flex: 1; }
.episode-badge       { font-size: 10px; font-weight: 600; padding: 1px 6px; border-radius: 3px; }
.episode-badge--passed  { color: var(--green);  background: rgba(74,222,128,0.10); }
.episode-badge--failed  { color: var(--red);    background: rgba(248,113,113,0.10); }
.episode-badge--active  { color: var(--accent); background: rgba(16,217,160,0.10); }
.episode-body        { padding: 4px 12px 12px 24px; display: flex; flex-direction: column; gap: 8px; }
.episode-body--empty { font-size: 11px; color: var(--text-3); padding: 4px 12px 10px 24px; }
.episode-field-label { font-size: 10px; font-weight: 700; text-transform: uppercase;
                       letter-spacing: 0.06em; color: var(--text-3); margin-bottom: 2px; }
.episode-field-item  { font-size: 12px; color: var(--text-1); line-height: 1.5; padding: 1px 0; }
.episode-field-item--decision { color: var(--cyan); }
.episode-field--rejected .episode-field-label { color: var(--red); }
.episode-field--rejected .episode-field-item  { color: var(--text-2); }
```

---

#### 7.13.6 DecisionNarrativeList — Choices & Why

**Source:** `progress.decisions` (current progress, not snapshot — shows the full accumulated decisions)

This is the most important section for non-developers. Every decision must be rendered as a complete sentence explaining:
1. **What** was chosen
2. **What the alternative was** (if any)
3. **Why** this choice was made
4. **When to revisit** (if specified)

**`decisionToSentence(d: DecisionItem): string`** — authoritative function in `explainer.ts`:

```typescript
export function decisionToSentence(d: DecisionItem): string {
  const chose    = d.chose.trim();
  const rejected = d.rejected?.trim();
  const reason   = d.reason?.trim();
  const revisit  = d.revisitIf?.trim();

  let sentence = '';

  if (rejected) {
    sentence = `**${chose}** was chosen over **${rejected}**`;
  } else {
    sentence = `The AI decided to use **${chose}**`;
  }

  if (reason) {
    sentence += ` because ${reason}`;
  }

  sentence += '.';

  if (revisit) {
    sentence += ` This may be reconsidered if ${revisit}.`;
  }

  return sentence;
}
```

**Rendering:**

```tsx
{progress?.decisions.length > 0 && (
  <section className="narrative-section">
    <div className="narrative-section-label">Choices & Why</div>
    <div className="narrative-decisions">
      {progress.decisions.map((d, i) => (
        <div key={i} className="narrative-decision-card">
          <div className="narrative-decision-chose">
            <span className="narrative-decision-tag">Chose</span>
            <span className="narrative-decision-value">{d.chose}</span>
            {d.rejected && (
              <>
                <span className="narrative-decision-vs">over</span>
                <span className="narrative-decision-rejected">{d.rejected}</span>
              </>
            )}
          </div>
          {d.reason && (
            <div className="narrative-decision-reason">
              <span className="narrative-decision-reason-icon">💡</span>
              <span>{d.reason}</span>
            </div>
          )}
          {d.revisitIf && (
            <div className="narrative-decision-revisit">
              Reconsider if: {d.revisitIf}
            </div>
          )}
        </div>
      ))}
    </div>
  </section>
)}
```

```css
.narrative-decisions          { display: flex; flex-direction: column; gap: 8px; }
.narrative-decision-card      { background: var(--bg-2); padding: 10px 12px; border-left: 2px solid var(--cyan); }
.narrative-decision-chose     { display: flex; align-items: center; gap: 6px; flex-wrap: wrap; margin-bottom: 4px; }
.narrative-decision-tag       { font-size: 9px; font-weight: 700; text-transform: uppercase;
                                 color: var(--text-3); letter-spacing: 0.08em; }
.narrative-decision-value     { font-size: 12px; font-weight: 600; color: var(--cyan); }
.narrative-decision-vs        { font-size: 10px; color: var(--text-3); }
.narrative-decision-rejected  { font-size: 12px; color: var(--text-2); text-decoration: line-through; }
.narrative-decision-reason    { font-size: 11px; color: var(--text-1); display: flex; gap: 6px;
                                 align-items: flex-start; line-height: 1.5; }
.narrative-decision-reason-icon { flex-shrink: 0; }
.narrative-decision-revisit   { font-size: 10px; color: var(--text-2); margin-top: 4px;
                                 padding-top: 4px; border-top: 1px solid var(--border); }
```

**Invariants:**
- Every decision must show `chose` + `reason` at minimum.
- If `reason === null`: card still renders but omits the reason row (no empty `💡` line).
- If `decisions.length === 0`: section not rendered.

---

#### 7.13.7 UncertaintyList — Open Questions

**Source:** `progress.uncertainties`

```tsx
{progress?.uncertainties.length > 0 && (
  <section className="narrative-section">
    <div className="narrative-section-label">
      Open Questions ({progress.uncertainties.length})
    </div>
    <div className="narrative-uncertainty-intro">
      The AI flagged {progress.uncertainties.length === 1 ? 'this question' : 'these questions'} as unresolved:
    </div>
    {progress.uncertainties.map((item, i) => (
      <div key={i} className="narrative-uncertainty-item">
        <span className="narrative-uncertainty-bullet">?</span>
        <span>{item}</span>
      </div>
    ))}
  </section>
)}
```

```css
.narrative-uncertainty-intro  { font-size: 11px; color: var(--text-2); margin-bottom: 6px; }
.narrative-uncertainty-item   { display: flex; gap: 8px; align-items: flex-start;
                                 font-size: 12px; color: var(--text-1); padding: 3px 0; }
.narrative-uncertainty-bullet { color: var(--yellow); font-weight: 700; flex-shrink: 0; width: 10px; }
```

---

#### 7.13.8 JudgeNarrativeList — Quality Check History

**Source:** `judgeFeedback[]`

Render only when `judgeFeedback.length > 0`. Show newest first.

```tsx
{judgeFeedback.length > 0 && (
  <section className="narrative-section">
    <div className="narrative-section-label">Quality Check History</div>
    {[...judgeFeedback].reverse().map((fb) => (
      <div key={fb.id} className={`narrative-judge-row narrative-judge-row--${fb.approved ? 'passed' : 'failed'}`}>
        <span className="narrative-judge-icon">{fb.approved ? '✓' : '✗'}</span>
        <div className="narrative-judge-body">
          <span className="narrative-judge-round">
            {fb.iteration !== null ? `Round ${fb.iteration}` : 'Unknown round'}
          </span>
          <span className="narrative-judge-verdict">
            {fb.approved ? 'Quality check passed.' : `Quality check flagged an issue${fb.summary ? ':' : '.'}`}
          </span>
          {!fb.approved && fb.summary && (
            <span className="narrative-judge-summary">"{fb.summary}"</span>
          )}
        </div>
      </div>
    ))}
  </section>
)}
```

```css
.narrative-judge-row          { display: flex; gap: 10px; align-items: flex-start; padding: 8px 0;
                                 border-bottom: 1px solid var(--border); }
.narrative-judge-row:last-child { border-bottom: none; }
.narrative-judge-icon         { font-size: 13px; font-weight: 700; flex-shrink: 0; margin-top: 1px; }
.narrative-judge-row--passed .narrative-judge-icon  { color: var(--green); }
.narrative-judge-row--failed .narrative-judge-icon  { color: var(--red); }
.narrative-judge-body         { display: flex; flex-direction: column; gap: 2px; }
.narrative-judge-round        { font-size: 10px; font-weight: 700; color: var(--text-3); text-transform: uppercase; letter-spacing: 0.06em; }
.narrative-judge-verdict      { font-size: 12px; color: var(--text-1); }
.narrative-judge-summary      { font-size: 11px; color: var(--text-2); font-style: italic; line-height: 1.4; }
```

---

#### 7.13.9 Empty State

Shown when `progress === null` AND `iterations.length === 0`:

```tsx
<div className="memory-browser">
  <div className="empty-state">
    <div className="empty-state-text">Start a loop to see the project story here.</div>
  </div>
</div>
```

---

#### 7.13.10 Full Component Shell

```tsx
// src/components/memory/MemoryBrowser.tsx
export function MemoryBrowser() {
  const { progress, iterations, judgeFeedback, loopState } = useStore();

  const hasContent = progress || iterations.length > 0;

  if (!hasContent) return <EmptyNarrativeState />;

  // Sort episodes newest-first
  const episodes = [...iterations].sort((a, b) => b.iteration - a.iteration);

  return (
    <div className="memory-browser narrative-memory">
      <GoalCard             progress={progress} />
      <ProgressCard         progress={progress} loopState={loopState} />
      <FocusCard            progress={progress} loopState={loopState} />
      <EpisodeList          episodes={episodes} feedback={judgeFeedback} />
      <DecisionNarrativeList progress={progress} />
      <UncertaintyList      progress={progress} />
      <JudgeNarrativeList   feedback={judgeFeedback} />
    </div>
  );
}
```

**CSS root:**

```css
.narrative-memory         { padding: 8px 0; }
.narrative-section        { padding: 10px 12px; border-bottom: 1px solid var(--border); }
.narrative-section:last-child { border-bottom: none; }
.narrative-section-label  {
  font-size: 10px; font-weight: 700; text-transform: uppercase;
  letter-spacing: 0.08em; color: var(--text-3); margin-bottom: 6px;
}
.narrative-goal-text      { font-size: 13px; color: var(--text-0); line-height: 1.6; }
.narrative-round-count    { font-size: 12px; color: var(--text-1); }
.narrative-progress-row   { display: flex; flex-direction: column; gap: 6px; }
```

---

#### 7.13.10b RubricViewer — Placement After Rewrite

The original `MemoryBrowser` rendered `<RubricViewer />` at the bottom. The §7.13 rewrite must preserve this.

Add as the last section inside `MemoryBrowser`, after `JudgeNarrativeList` and before `JudgeEvolutionLog` (§7.7):

```tsx
{rubricsContent && (
  <section className="narrative-section">
    <div className="narrative-section-label">Quality Rubric</div>
    <RubricViewer />
  </section>
)}
```

- Only render when `rubricsContent !== null` (same guard as before).
- `RubricViewer` is an existing component — no changes to it required.

---

#### 7.13.11 `explainer.ts` — Required New Exports

Add these alongside existing exports. Do not remove existing functions.

```typescript
// NEW exports to add to src/lib/explainer.ts

/** Convert a DecisionItem to a plain-English sentence. */
export function decisionToSentence(d: DecisionItem): string { /* see §7.13.6 */ }

/** Map a confidence number (30–100) to a plain-English label. */
export function confidenceLabel(n: number): string { /* see §7.13.3 */ }
```

---

### 7.14 Terminal Output Persistence Per Project

**Problem:** The xterm.js terminal is a single shared instance. When the user closes and reopens the terminal panel, or switches between projects, the output is lost — xterm's internal scrollback is cleared on `reset()`. There is no way to recover what Claude printed.

**Solution:** A per-project output buffer in the store that captures every PTY chunk. On project switch or panel reopen, replay the buffer into xterm instead of starting blank.

---

#### 7.14.1 Buffer Store (settingsSlice or new terminalSlice)

```typescript
// Add to store — keyed by projectId
interface TerminalBufferSlice {
  terminalBuffers: Record<string, string[]>;  // projectId → ordered chunks
  bufferAdd:    (projectId: string, chunk: string) => void;
  bufferReplay: (projectId: string) => string[];  // returns chunks in order
  bufferClear:  (projectId: string) => void;
}
```

**Buffer cap:** maximum **1000 chunks per project**. When full, drop the oldest chunk before appending (ring buffer semantics):

```typescript
bufferAdd: (projectId, chunk) => set(state => {
  const existing = state.terminalBuffers[projectId] ?? [];
  const next = existing.length >= 1000
    ? [...existing.slice(1), chunk]
    : [...existing, chunk];
  return { terminalBuffers: { ...state.terminalBuffers, [projectId]: next } };
}),
```

**`bufferReplay`** returns the stored array; the caller writes each chunk to xterm in sequence.

---

#### 7.14.2 Three Integration Points

**Point 1 — On output received (`claude-output` listener in App.tsx):**

Capture every chunk before writing to xterm. The `activeProjectId` at the time of the event is the key.

```typescript
listen<{ project_id: string; data: string }>('claude-output', (event) => {
  const { project_id, data } = event.payload;

  // 1. Buffer first
  useStore.getState().bufferAdd(project_id, data);

  // 2. Then write to terminal (only if it's the active project)
  if (project_id === useStore.getState().activeProjectId) {
    terminalRef.current?.write(data);
  }
});
```

**Point 2 — On project switch (when `activeProjectId` changes):**

Reset xterm and replay the buffer for the newly selected project:

```typescript
// Called wherever activeProjectId is updated (ProjectList click handler or store action)
function onProjectSwitch(newProjectId: string) {
  setActiveProject(newProjectId, project.path);
  const term = terminalRef.current;
  if (!term) return;

  term.reset();  // clear current display

  const chunks = useStore.getState().bufferReplay(newProjectId);
  for (const chunk of chunks) {
    term.write(chunk);
  }
}
```

**Point 3 — On session dead (`claude-session-dead` event):**

Append a dim visual separator to the buffer so the user can see where the session ended when they reopen the panel:

```typescript
listen<{ project_id: string }>('claude-session-dead', (event) => {
  const separator = '\r\n\x1b[2m--- session ended ---\x1b[0m\r\n';
  useStore.getState().bufferAdd(event.payload.project_id, separator);

  // Write to terminal immediately if this is the active project
  if (event.payload.project_id === useStore.getState().activeProjectId) {
    terminalRef.current?.write(separator);
  }
});
```

`\x1b[2m` = dim text, `\x1b[0m` = reset. The separator is stored as a regular chunk so it appears on replay.

---

#### 7.14.3 Terminal Panel Reopen

When `showTerminal` toggles from `false` → `true` and the terminal component mounts (or becomes visible), replay the buffer for the active project:

```typescript
// In the terminal component — useEffect on mount / visibility change
useEffect(() => {
  if (!showTerminal || !terminalRef.current) return;
  const projectId = useStore.getState().activeProjectId;
  if (!projectId) return;

  const term = terminalRef.current;
  term.reset();
  const chunks = useStore.getState().bufferReplay(projectId);
  for (const chunk of chunks) {
    term.write(chunk);
  }
}, [showTerminal]);
```

---

#### 7.14.4 Buffer Lifecycle

| Event | Action |
|-------|--------|
| New session spawned (`spawn_and_send` / `spawn_claude`) | `bufferClear(projectId)` — start fresh |
| PTY output chunk received | `bufferAdd(projectId, chunk)` |
| Session dead | `bufferAdd(projectId, separator)` |
| Project deleted | `bufferClear(projectId)` |
| `resetLoop()` called | `bufferClear(activeProjectId)` |

**Buffer is NOT cleared on:**
- Terminal panel close/reopen
- Project switch (buffer must survive to replay on switch-back)
- App minimize/restore

---

#### 7.14.5 File Scope

| File | Change |
|------|--------|
| `src/store/settingsSlice.ts` (or new `terminalSlice.ts`) | Add `terminalBuffers`, `bufferAdd`, `bufferReplay`, `bufferClear` |
| `src/App.tsx` | `claude-output` listener calls `bufferAdd`; `claude-session-dead` appends separator |
| `src/components/layout/TerminalPanel.tsx` (or equivalent) | `useEffect` on mount/show calls `bufferReplay` + writes chunks to xterm |
| `src/components/projects/ProjectList.tsx` | `onProjectSwitch` calls `term.reset()` + `bufferReplay` |

---

## 8. UI / Layout Specifications

### 8.1 Titlebar

**File:** `src/components/layout/Titlebar.tsx`

Replace all text buttons with Lucide icon buttons. No visible text labels.

```tsx
import { PanelRight, TerminalSquare } from 'lucide-react';

// Replace text buttons:
<button className="icon-btn titlebar-icon-btn" onClick={toggleRightPanel} title="Toggle panel">
  <PanelRight size={14} strokeWidth={1.8} />
</button>
<button className="icon-btn titlebar-icon-btn" onClick={toggleTerminal} title="Toggle terminal">
  <TerminalSquare size={14} strokeWidth={1.8} />
</button>
```

```css
.titlebar-icon-btn { width: 28px; height: 28px; -webkit-app-region: no-drag; }
```

**Test:** After change, `Titlebar` must contain zero visible text nodes in the button area.

---

### 8.2 Right Panel — Vertical Activity Bar

**File:** `src/components/layout/RightPanel.tsx`

**Replace** horizontal pill tabs with a vertical icon activity bar (left side of right panel).

```tsx
// src/components/layout/RightPanel.tsx

type RightTabId = 'activity' | 'memory' | 'settings';

const TABS: { id: RightTabId; icon: ReactNode; label: string }[] = [
  { id: 'activity', icon: <Zap size={15} />,          label: 'Activity' },
  { id: 'memory',   icon: <BrainCircuit size={15} />,  label: 'Memory' },
  { id: 'settings', icon: <Settings2 size={15} />,     label: 'Settings' },
];

export function RightPanel() {
  const { rightTab, setRightTab, showRightPanel, setShowRightPanel } = useStore();

  const handleTabClick = (id: RightTabId) => {
    if (rightTab === id && showRightPanel) {
      setShowRightPanel(false);
    } else {
      setRightTab(id);
      setShowRightPanel(true);
    }
  };

  return (
    <div className="right-panel-inner">
      {/* Always-visible vertical bar */}
      <div className="right-panel-activity-bar">
        {TABS.map(({ id, icon, label }) => (
          <button
            key={id}
            className={`activity-tab${rightTab === id && showRightPanel ? ' active' : ''}`}
            onClick={() => handleTabClick(id)}
            title={label}
          >
            {icon}
          </button>
        ))}
      </div>

      {/* Toggleable content panel */}
      {showRightPanel && rightTab && (
        <div className="right-panel-content">
          <div className="right-panel-content-header">
            <span className="right-panel-content-title">
              {TABS.find(t => t.id === rightTab)?.label}
            </span>
            <button className="icon-btn" onClick={() => setShowRightPanel(false)}>
              <X size={12} />
            </button>
          </div>
          <div className="right-panel-scroll">
            {rightTab === 'activity' && <ActivityFeed />}
            {rightTab === 'memory'   && <MemoryBrowser />}
            {rightTab === 'settings' && <ProjectSettings />}
          </div>
        </div>
      )}
    </div>
  );
}
```

**Layout measurements:**
- Activity bar: `36px` wide, always visible, `border-right: 1px solid var(--border)`
- Tab button: `28×28px`, centered icon
- Active tab: `2px solid var(--accent)` left border + `var(--accent-glow)` background
- Content panel: `280px` wide, appears to the left of activity bar

**`AppShell.tsx` changes:**
- Remove `rightPanel` prop conditional rendering.
- Render `<RightPanel />` directly inside `.app-body` — activity bar is always present.

---

### 8.3 ProjectCard → Tree Row Pattern

**File:** `src/components/projects/ProjectCard.tsx`

**Replace** card-style layout with compact tree-row pattern:

```tsx
<div
  className={`tree-row${active ? ' tree-row--active' : ''}`}
  onClick={onClick}
>
  <span
    className="tree-row-dot"
    style={{ background: statusDotColor(project.status) }}
  />
  <span className="tree-row-label">{project.name}</span>

  <div className="tree-row-actions">
    {hasActiveSession && (
      <button
        className="tree-action-btn tree-action-btn--danger"
        onClick={handleKill}
        title="Kill session"
      >
        <Square size={11} />
      </button>
    )}
    <button
      className="tree-action-btn tree-action-btn--danger"
      onClick={handleDeleteClick}
      title="Delete"
    >
      <Trash2 size={11} />
    </button>
  </div>

  {badge && (
    <span className="tree-row-badge" style={{ color: badge.dot }}>
      {badge.label}
    </span>
  )}
</div>
```

```typescript
function statusDotColor(status: ProjectStatus): string {
  return {
    running:   'var(--green)',
    paused:    'var(--yellow)',
    completed: 'var(--accent)',
    cancelled: 'var(--red)',
    idle:      'var(--text-3)',
  }[status];
}
```

**CSS:**

```css
.tree-row {
  display: flex; align-items: center; gap: 6px;
  height: 26px; padding: 0 8px 0 10px; cursor: pointer;
  transition: background var(--t-fast);
  position: relative;
}
.tree-row:hover { background: var(--bg-hover); }
.tree-row:hover .tree-row-actions  { opacity: 1; pointer-events: auto; }
.tree-row:hover .tree-row-badge    { opacity: 0; }

.tree-row--active {
  background: var(--accent-glow);
}
.tree-row--active::before {
  content: ''; position: absolute; left: 0; top: 0; bottom: 0;
  width: 2px; background: var(--accent);
}

.tree-row-dot   { width: 6px; height: 6px; border-radius: 50%; flex-shrink: 0; }
.tree-row-label { flex: 1; font-size: 13px; color: var(--text-1);
                  overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.tree-row-badge { font-size: 10px; color: var(--text-2); }

.tree-row-actions {
  display: flex; gap: 2px; opacity: 0; pointer-events: none;
  transition: opacity var(--t-fast);
}
.tree-action-btn {
  width: 18px; height: 18px; display: flex; align-items: center; justify-content: center;
  background: none; border: none; cursor: pointer; color: var(--text-2);
  transition: color var(--t-fast);
}
.tree-action-btn:hover { color: var(--text-0); }
.tree-action-btn--danger:hover { color: var(--red); }
```

---

## 9. File Scope

All files that must be created or modified:

| File | Change | Key additions |
|------|--------|---------------|
| `src/types/loop.ts` | Modify | `ActivityEventType`, `ActivityEvent`, `ProgressSnapshot` types |
| `src/store/loopSlice.ts` | Modify | `rawProgressContent`, `activityEvents`, `addActivityEvent`; update `setProgress` signature; ActivityEvent generation logic in `setLoopState`, `setJudgeFeedback` |
| `src/store/settingsSlice.ts` | Modify | `activeQuestion`, `setActiveQuestion` |
| `src/App.tsx` | Modify | Update `setProgress(parsed, raw)` call; add `judge-evolution-changed` listener if missing; `claude-output` listener calls `bufferAdd`; `claude-session-dead` appends separator (§7.14.2) |
| `src/lib/state-parser.ts` | Verify | Ensure phase derivation matches spec §3.1 |
| `src/lib/progress-parser.ts` | Verify/Modify | Ensure `ORIGINAL_GOAL`, `ITERATION`, `revisit_if` fields are extracted; ensure `CONFIDENCE` clamping |
| `src/lib/rubric-generator.ts` | Modify | Default `maxIterations=50`, anti-early-completion instructions |
| `src/components/monitor/ActivityFeed.tsx` | **New** | Full implementation per §7.1 |
| `src/components/monitor/IterationTimeline.tsx` | Modify | Rich snapshot panel per §7.2 |
| `src/components/monitor/ProgressSummary.tsx` | Modify | Expand/collapse per §7.3; null guard |
| `src/components/monitor/JudgeFeedback.tsx` | Modify | `formatFeedbackTime`; iteration badge per §7.4 |
| `src/components/monitor/LoopMonitor.tsx` | Modify | SessionReport per §7.5; render `QuestionPromptOverlay` |
| `src/components/monitor/QuestionPromptOverlay.tsx` | **New** | Full implementation per §7.8 |
| `src/components/projects/ProjectList.tsx` | Modify | Completed project history view per §7.6 |
| `src/components/memory/MemoryBrowser.tsx` | **Rewrite** | Narrative panel per §7.13; `JudgeEvolutionLog` at bottom per §7.7 |
| `src/lib/explainer.ts` | Modify | Add `decisionToSentence()`, `confidenceLabel()` (§7.13.11) |
| `src/components/layout/RightPanel.tsx` | Modify | Vertical activity bar per §8.2; raw log tab for `advancedMode` |
| `src/components/layout/Titlebar.tsx` | Modify | Icon-only buttons per §8.1 |
| `src/components/projects/ProjectCard.tsx` | Modify | Tree row pattern per §8.3 |
| `src/styles/globals.css` | Modify | Add: `snapshot-*`, `activity-event-*`, `result-stat-*`, `question-*`, `tree-row-*`, `activity-tab`, `right-panel-activity-bar`, `narrative-*`, `confidence-bar-*`, `episode-*` classes |
| `src/store/settingsSlice.ts` (or `terminalSlice.ts`) | Modify | `terminalBuffers`, `bufferAdd`, `bufferReplay`, `bufferClear` (§7.14.1) |
| `src/components/layout/TerminalPanel.tsx` | Modify | `useEffect` mount/show → `bufferReplay` + write to xterm (§7.14.3) |
| `src/lib/cost-calculator.ts` | Modify | Delete `formatElapsedMinutes`; keep `estimateCostKrw`, `formatCostKrw` (§7.12.4) |
| `src/components/monitor/TokenUsageCard.tsx` | Modify | Add 30s interval for live re-render (§7.12.3); null guards per §7.12.5 |
| `src/components/monitor/LoopMonitor.tsx` | Modify | SessionReport stats row: add Elapsed + Est. cost cards (§7.12.6) |
| `src-tauri/src/watcher/mod.rs` | Verify | Confirm `JUDGE_EVOLUTION.md` → `judge-evolution-changed` is implemented |
| `src-tauri/src/process/mod.rs` | Modify | `detect_question_block`, `QuestionBlock` struct, emit `claude-question-prompt`; `ready: Arc<AtomicBool>` in `ClaudeProcess`; `is_claude_prompt()`, `strip_ansi()`, `is_session_ready()` (§7.11.1) |
| `src/lib/rubric-generator.ts` | Modify | Flatten multi-line prompt (§7.10); defaults `maxIterations=50`, `milestoneEvery=0` |
| `src-tauri/src/lib.rs` | Modify | `spawn_and_send`: streaming-ready wait + 2-phase send (§7.11.1) |
| `src/components/projects/QuickRestartView.tsx` | Modify | `setActiveProject` before `start_watcher` (§7.11.4); `invoke('spawn_and_send', ...)` (§7.11.1); flatten prompt |
| `src/components/wizard/GoalWizard.tsx` | Modify | `setActiveProject` before `start_watcher` (§7.11.4); `invoke('spawn_and_send', ...)` (§7.11.1); flatten prompt |
| `src-tauri/src/process/mod.rs` | Modify | `send_command_enter`: single `write_all("{data}\r")`, no `\n` (§7.11.5) |
| `src/components/monitor/MilestoneGate.tsx` | Modify | Add `projectId`; use `send_command_enter` (§7.11.2, §7.11.3) |
| `src/components/monitor/HITLGate.tsx` | Modify | Unify to `send_command_enter` with `projectId` (§7.11.3) |

---

## 10. Acceptance Criteria (Binary Pass/Fail)

Every item must pass before the feature is considered complete.

### Parser Unit Tests

```
[ ] P1.  parseProgress("COMPLETED:\n- task 1\n- task 2\nCONFIDENCE: 72\nNEXT_STEP: next")
         → completed=["task 1","task 2"], confidence=72, nextStep="next"

[ ] P2.  parseProgress("") → returns null

[ ] P3.  parseProgress("CONFIDENCE: 150") → confidence=100

[ ] P4.  parseProgress("CONFIDENCE: 10")  → confidence=30

[ ] P5.  parseProgress("CONFIDENCE: abc") → confidence=null

[ ] P6.  parseProgress("DECISIONS:\n- chose: React, rejected: Vue, reason: ecosystem")
         → decisions=[{chose:"React", rejected:"Vue", reason:"ecosystem", revisitIf:null}]

[ ] P7.  parseProgress("DECISIONS:\n- chose: TypeScript")
         → decisions=[{chose:"TypeScript", rejected:null, reason:null, revisitIf:null}]

[ ] P8.  parseLoopState(yaml with active:true, pause_reason:"") → phase="running"

[ ] P9.  parseLoopState(yaml with active:false, pause_reason:"hitl") → phase="paused"

[ ] P10. parseLoopState(yaml with active:false, pause_reason:"milestone") → phase="paused"

[ ] P11. parseLoopState(yaml with active:false, pause_reason:"") → phase="completed"

[ ] P12. parseLoopState(null) → null (no exception)

[ ] P13. parseJudgeFeedback("## Iteration 3 — REJECTED (2 consecutive) — 2026-01-01 12:00:00\n\nBad code.")
         → [{iteration:3, approved:false, consecutive:2, summary:"Bad code."}]

[ ] P14. JUDGE_EVOLUTION.md line not matching regex → silently skipped, no exception
```

### State Machine

```
[ ] S1.  loop-state-changed(active:true) → loopState.phase === 'running'

[ ] S2.  loop-state-changed(active:false, pause_reason:'hitl') → phase === 'paused'

[ ] S3.  Sequence: paused → loop-state-changed(active:true) → phase === 'running'

[ ] S4.  loop-state-changed(active:false, pause_reason:'') → phase === 'completed'

[ ] S5.  loop-state-changed with deleted:true → phase === 'completed'

[ ] S6.  loop-state-changed({ content:null, deleted:false }) → loopState === null
         (deleted:true takes precedence → see S5; only deleted:false triggers null reset)

[ ] S7.  After phase='completed', another loop-state-changed does NOT change phase
         (terminal state invariant)
```

### ActivityFeed

```
[ ] A1.  First loop-state-changed(active:true, prev=null) → 'loop-started' event in activityEvents

[ ] A2.  loop-state-changed(iteration:2, prev.iteration:1) → 'iteration-complete' event

[ ] A3.  loop-state-changed(phase:'paused', pauseReason:'hitl') → 'loop-paused-hitl' event

[ ] A4.  loop-state-changed(phase:'paused', pauseReason:'milestone') → 'loop-paused-milestone' event

[ ] A5.  loop-state-changed(phase:'completed') → 'loop-completed' event

[ ] A6.  setProgress(data, raw) → 'progress-updated' event

[ ] A7.  JudgeFeedbackEntry with approved:false added → 'iteration-failed' event

[ ] A8.  ActivityFeed renders events newest-first

[ ] A9.  ActivityFeed with 0 events → renders "Start a loop to see activity." and nothing else

[ ] A10. 'iteration-complete' dot color = var(--green)

[ ] A11. 'iteration-failed' dot color = var(--red)

[ ] A12. 'loop-completed' dot color = var(--accent)

[ ] A13. Sequence: phase=paused → loop-state-changed(active:true) → 'loop-resumed' event in activityEvents;
         dot color = var(--cyan)

[ ] A14. loop-state-changed(phase:'cancelled') → 'loop-cancelled' event in activityEvents;
         dot color = var(--red)
```

### IterationTimeline

```
[ ] T1.  Iteration count goes 1→2 → new node "Round 2" appears in timeline

[ ] T2.  Active node sublabel = progress.nextStep value

[ ] T3.  nextStep === null → no sublabel rendered (no empty string, no placeholder)

[ ] T4.  Click on round with snapshot → snapshot panel shows completed/decisions/uncertainties

[ ] T5.  Click on round without snapshot → "No snapshot for this round." rendered

[ ] T6.  Round with REJECTED JudgeFeedback → node has red border (failed status)

[ ] T7.  Round with both REJECTED and APPROVED entries → node is 'completed' (not 'failed')

[ ] T8.  snapshot.confidence === null → confidence display is absent (no "null%" or "0%")

[ ] T9.  snapshot.completed.length === 0 → completed section not rendered at all

[ ] T10. loop completes (phase→'completed') → final round node in IterationTimeline
         has a progressSnapshot (not undefined); clicking it shows "What was built"
         (tests §6.3 terminal snapshot capture)
```

### SessionReport

```
[ ] R1.  loopState.phase === 'completed' → SessionReport rendered

[ ] R2.  loopState.phase === 'cancelled' → SessionReport rendered with "Remaining" section

[ ] R3.  loopState.phase === 'running' → SessionReport NOT rendered

[ ] R4.  loopState.phase === 'cancelled', progress.remainingGap=[] → "Remaining" section not rendered

[ ] R5.  progress.completed.length === 5 → first 3 shown + "+2 more" button

[ ] R6.  Click "+2 more" → all 5 shown + "Show less" button

[ ] R7.  judgeEnabled:false → no "Rejections" stat card

[ ] R8.  judgeEnabled:true → "Rejections" stat card visible

[ ] R9.  progress.originalGoal === null → GoalBanner shows "No goal recorded" (not empty)

[ ] R10. "New Goal" button calls resetLoop() and transitions to GoalWizard
```

### Completed Project History

```
[ ] H1.  project.status==='completed' + session data exists → click shows SessionReport

[ ] H2.  project.status==='completed' OR 'cancelled' + no session data → click shows QuickRestartView

[ ] H3.  project.status==='running' → click shows LoopMonitor (running view)

[ ] H4.  synthetic LoopState.iteration = max(iterations[].iteration) from session data

[ ] H5.  synthetic LoopState.originalPrompt = session data progress.originalGoal
```

### UI / Layout

```
[ ] U1.  ProjectCard height = 26px (measured via DevTools computed styles)

[ ] U2.  tree-row hover: action buttons (Trash2, Square) appear; badge disappears

[ ] U3.  tree-row normal state: action buttons hidden (opacity:0, pointer-events:none)

[ ] U4.  Titlebar: zero text nodes in button area (icon-only)

[ ] U5.  Right panel activity bar (36px wide) always visible regardless of rightTab state

[ ] U6.  Same-tab click when panel open → panel closes; activity bar remains visible

[ ] U7.  Right panel content area width = 280px

[ ] U8.  Active activity-tab has 2px left accent border + accent-glow background
```

### AskUserQuestion Overlay

```
[ ] Q1.  PTY outputs "Your choice (1-3):" pattern → overlay appears

[ ] Q2.  Overlay shows question text and all choices

[ ] Q3.  Clicking choice 2 → send_command called with "2\r" → overlay closes within 200ms

[ ] Q4.  "Type manually" click → overlay closes, terminal remains usable

[ ] Q5.  choices.length > 10 → overlay shows max 10 choices

[ ] Q6.  loopState === null → overlay not shown even if event fires
```

### PTY Transmission & Timing (Bug Fixes §7.10–7.11)

```
[ ] G1.  buildGulfLoopPrompt output has all \n replaced with space before PTY send
         — verify: the `command` argument passed to invoke('spawn_and_send') contains no \n or \r
         (the flat prompt goes to spawn_and_send, NOT to send_command_enter)

[ ] G2.  QuickRestartView and GoalWizard call invoke('spawn_and_send') — NOT
         invoke('spawn_claude') followed by a separate invoke('send_command_enter')

[ ] G3.  No frontend setTimeout, clearTimeout, or claude-output listener is created
         during loop start (verify: no listen('claude-output') call in start handlers)

[ ] G4.  spawn_and_send uses tokio::time::sleep (not std::thread::sleep) — verify
         by confirming PTY output events (claude-output) still arrive during the
         streaming wait period; these events are required for readiness detection

[ ] G5.  Component unmount during 8s wait does not cause double-send or JS error
         (the invoke Promise may resolve after unmount — must be handled safely)

[ ] G6.  MilestoneGate "Resume" button calls invoke('send_command_enter', { projectId, data })
         — projectId must NOT be undefined or empty string

[ ] G7.  MilestoneGate "Cancel" button calls invoke('send_command_enter', { projectId, data })

[ ] G8.  HITLGate "Resume" button calls invoke('send_command_enter', { projectId, data })

[ ] G9.  HITLGate "Cancel" button calls invoke('send_command_enter', { projectId, data })

[ ] G10. No gate component uses bare invoke('send_command', ...) for slash commands

[ ] G11. setActiveProject(projectId, projectPath) is called synchronously BEFORE
         invoke('start_watcher') and invoke('spawn_and_send') in QuickRestartView and GoalWizard

[ ] G12. While spawn_and_send is in the streaming wait, loop-state-changed events are
         routed to the correct project (verify: activeProjectId is non-null during the wait)

[ ] G13. send_command_enter writes "{data}\r" as a single write_all call — no \n appended,
         no two separate writes

[ ] G14. After sending the gulf-loop command, Claude executes it (LoopMonitor transitions
         from StartingScreen to running view within ~10s of spawn_and_send resolving)

[ ] G15. ClaudeProcess struct has a `ready: Arc<AtomicBool>` field; it is reset to false
         when spawn_claude_inner creates a new session

[ ] G16. PTY reader thread calls is_claude_prompt() on each output chunk;
         when "> " or "❯ " is detected, ready is set to true via AtomicBool::store

[ ] G17. spawn_and_send does NOT enter Phase 1 until either:
         (a) ready=true AND elapsed >= 3s, followed by 300ms buffer, OR
         (b) elapsed >= 15s (hard timeout)
         — verify by artificially delaying Claude init and confirming command still executes

[ ] G18. Rust PTY write sequence in spawn_and_send:
         (1) write_all(command.as_bytes()) — no \r
         (2) tokio::time::sleep(500ms)
         (3) write_all(b"\r") — Enter only
         Verify: log raw bytes written; confirm \r is a separate write from command text
```

### Usage Tracking (§7.12)

```
[ ] C1.  TokenUsageCard renders "N / M 라운드" + progress bar when loopState !== null

[ ] C2.  TokenUsageCard renders "약 ₩N,NNN (추정)" using estimateCostKrw(iteration)

[ ] C3.  TokenUsageCard renders elapsed time using formatElapsed(loopState.startedAt)

[ ] C4.  loopState.startedAt === null → elapsed time row absent (no "0분 경과")

[ ] C5.  loopState === null → TokenUsageCard renders nothing

[ ] C6.  Elapsed time updates on screen without requiring a loop state change:
         wait 30s during a running loop → elapsed label increments

[ ] C7.  The 30s interval does NOT run when phase is paused, completed, or cancelled

[ ] C8.  SessionReport stats row shows Elapsed and Est. cost cards alongside Rounds/Tasks/Confidence

[ ] C9.  SessionReport: loopState.startedAt === null → Elapsed card not rendered

[ ] C10. formatElapsedMinutes is deleted from cost-calculator.ts (or not exported/used)

[ ] C11. loopStartTimeMs is not used as a display source anywhere in the UI
         (it may remain in session persistence only)
```

### Terminal Persistence (§7.14)

```
[ ] X1.  After loop produces output, close the terminal panel and reopen it →
         all previously seen output is visible again (no blank terminal)

[ ] X2.  Switch to a different project and back → original project's terminal
         output is restored (term.reset() + bufferReplay called on switch)

[ ] X3.  claude-session-dead fires → "--- session ended ---" appears in dim text
         at the bottom of the terminal; visible on next replay

[ ] X4.  bufferAdd respects 1000-chunk cap: inject 1001 chunks → oldest chunk
         is dropped; bufferReplay returns exactly 1000 chunks

[ ] X5.  spawn_and_send (new session) → previous buffer for that projectId is
         cleared (bufferClear called before spawn); old output not shown

[ ] X6.  resetLoop() → bufferClear(activeProjectId) is called;
         terminal panel shows blank after reset

[ ] X7.  PTY output for non-active project → chunk stored in buffer but NOT
         written to xterm (no cross-project terminal bleed)
```

### Narrative Memory Panel (§7.13)

```
[ ] N1.  GoalCard renders progress.originalGoal text when non-null;
         section is absent when originalGoal === null

[ ] N2.  ConfidenceBar: confidence=72 → bar fill width=72%; label="More than halfway — major work completed"

[ ] N3.  ConfidenceBar: confidence=null → entire ConfidenceBar not rendered (no 0% placeholder)

[ ] N4.  FocusCard visible only when phase=running or phase=paused AND nextStep is non-null;
         not rendered when phase=completed, cancelled, or idle

[ ] N5.  EpisodeList renders newest iteration first (highest iteration number at top)

[ ] N6.  Collapsed episode row shows round number and badge only; no body content in DOM

[ ] N7.  Episode with wasPassed=true → badge text "✓ Passed", badge class episode-badge--passed

[ ] N8.  Episode with wasRejected=true AND wasPassed=false → badge text "✗ Rejected", class episode-badge--failed

[ ] N9.  Episode with no judgeFeedback entries → badge text "—"

[ ] N10. Episode expanded + progressSnapshot exists → "What was built" section shows
         snapshot.completed items; "Key decision" shows first decision as decisionToSentence()

[ ] N10b. Episode expanded + REJECTED judgeFeedback exists for that iteration →
          "Quality check issue" field shown with fb.summary text; label in var(--red)

[ ] N11. Episode expanded + progressSnapshot === undefined → "No details recorded for this round." shown

[ ] N12. DecisionNarrativeList: decision with chose="TypeScript", rejected="JavaScript", reason="type safety"
         → card renders "TypeScript" (cyan, weight 600) + strikethrough "JavaScript" + reason text

[ ] N13. DecisionNarrativeList: decision with reason=null → reason row absent (no empty 💡 line)

[ ] N14. DecisionNarrativeList: decisions.length===0 → "Choices & Why" section not rendered

[ ] N15. decisionToSentence({chose:"React", rejected:"Vue", reason:"ecosystem", revisitIf:null})
         → "**React** was chosen over **Vue** because ecosystem."
         (implementation may omit markdown bold — plain text form also acceptable in UI)

[ ] N16. UncertaintyList: uncertainties=["X","Y"] → intro text "The AI flagged these questions as unresolved:"
         + 2 items each with "?" bullet in var(--yellow)

[ ] N17. UncertaintyList: uncertainties.length===0 → "Open Questions" section not rendered

[ ] N18. JudgeNarrativeList: approved entry → "✓" icon (var(--green)) + "Quality check passed."

[ ] N19. JudgeNarrativeList: rejected entry with summary → "✗" icon (var(--red))
         + verdict text + summary in italic below

[ ] N20. JudgeNarrativeList: judgeFeedback.length===0 → "Quality Check History" section not rendered

[ ] N21. Empty state (progress===null AND iterations.length===0) →
         "Start a loop to see the project story here." — no other content
```

### Build Verification

```
[ ] B1.  npx tsc --noEmit — zero errors

[ ] B2.  cargo check — zero errors

[ ] B3.  Full gulf-loop run (5+ iterations): round counter increments in timeline

[ ] B4.  gulf-loop completes: SessionReport shows correct stats (rounds, tasks done, confidence)

[ ] B5.  Reload app, click completed project → SessionReport loads from session.json
```

---

## 11. Future Work

### Blocked / Needs Clarification

0. **Structured memory wiki watcher** (`structured_memory: true`) — If gulf-loop writes `.claude/memory/*.md` wiki files, the watcher and `MemoryBrowser` need a new watch target and `memory-wiki-changed` event. The narrative panel (§7.13) would show a "Knowledge Base" section. **Requires**: confirmation of exact file paths and format used when `structured_memory: true`. Until confirmed, §7.13 covers the existing 5-file data sources only.

### High Priority

1. **Gulf-loop install verification** — Before starting a loop, call `check_gulf_loop_installed()`. If `false`, show a clear error with install instructions. Block loop start.

2. **Max iterations UI** — Add a slider (`10–100`) in `QuickRestartView` that sets `maxIterations` when starting a new loop.

3. **Force-max toggle** — Checkbox in `QuickRestartView` that adds `force_max: true` to gulf-loop state file, preventing early completion.

4. **Session list view** — "View history" button per project. Calls `list_sessions(projectPath)` and displays archived sessions as selectable items.

5. **JudgeEvolution in right panel** — `JudgeEvolutionLog` component integrated into MemoryBrowser tab.

### Medium Priority

6. **Round-over-round progress diff** — When viewing a snapshot, show Δ confidence and Δ completed tasks vs. previous round.

7. **Estimated cost display** — `TokenUsageCard` should show a running cost estimate: `estimateCostKrw(loopState.iteration)` from `cost-calculator.ts`.

8. **Theme persistence fix** — `save_settings` must be called after every theme change. Verify with `get_settings` on restart.

### Lower Priority

9. **Offline cached sessions** — When `start_watcher` fails (project path inaccessible), auto-load the last `session.json` and show it as read-only.

10. **Multi-project comparison** — Side-by-side `SessionReport` view for two projects.

11. **Export session report** — "Export" button on `SessionReport` that serializes the full report to Markdown and opens a save dialog via Tauri's `dialog` plugin.
