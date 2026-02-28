# Pipeline Tracking — Unified Chapter Pipeline Run Record

## Problem

Pipeline status is currently scattered across:
- Two status columns on `novel_chapters` (`llm_processing_status`, `llm_narrative_status`)
- A separate `chapter_tts` table for TTS errors
- Logs for scraping and preprocessing failures (not queryable)
- `failed_jobs` table for catastrophic job crashes

After the pipeline restructure (3 LLM stages + new character status column), there will be 3+ status columns on `novel_chapters` and still no single place to ask: *"What happened to chapter X? What stage failed and why?"*

**Fix:** A single `chapter_pipeline_runs` row per chapter tracks every stage's status, timing, and failure reason in one place.

---

## Schema: `chapter_pipeline_runs`

One row per chapter (upsert on `chapter_id`). Each stage gets 4 columns: status, error, started_at, completed_at.

```
chapter_pipeline_runs
─────────────────────────────────────────────────────────────────────────
id
chapter_id           UNIQUE FK → novel_chapters (cascade delete)
overall_status       VARCHAR(20)  — pending | running | complete | failed
run_count            UNSIGNED INT DEFAULT 1  — increments on each reset/rerun

# Stage: Scrape
scrape_status        VARCHAR(20) DEFAULT 'pending'
scrape_error         TEXT nullable
scrape_started_at    TIMESTAMP nullable
scrape_completed_at  TIMESTAMP nullable

# Stage: Preprocess (rules + table detection)
preprocess_status    VARCHAR(20) DEFAULT 'pending'
preprocess_error     TEXT nullable
preprocess_started_at    TIMESTAMP nullable
preprocess_completed_at  TIMESTAMP nullable

# Pipeline 1: Character Association (NEW — from restructure plan)
characters_status    VARCHAR(20) DEFAULT 'pending'
characters_error     TEXT nullable
characters_started_at    TIMESTAMP nullable
characters_completed_at  TIMESTAMP nullable

# Pipeline 2: Dialogue Attribution
dialogue_status      VARCHAR(20) DEFAULT 'pending'
dialogue_error       TEXT nullable
dialogue_started_at      TIMESTAMP nullable
dialogue_completed_at    TIMESTAMP nullable

# Pipeline 3: Narrative Analysis
narrative_status     VARCHAR(20) DEFAULT 'pending'
narrative_error      TEXT nullable
narrative_started_at     TIMESTAMP nullable
narrative_completed_at   TIMESTAMP nullable

# TTS
tts_status           VARCHAR(20) DEFAULT 'pending'
tts_error            TEXT nullable
tts_started_at       TIMESTAMP nullable
tts_completed_at     TIMESTAMP nullable

created_at
updated_at
─────────────────────────────────────────────────────────────────────────
```

**Status values per stage:** `pending` | `running` | `complete` | `failed` | `skipped`

**`overall_status` logic:**
- `pending` — no stage has started
- `running` — at least one stage in progress (or some complete, not all)
- `complete` — all required stages complete (scrape + preprocess + characters + dialogue + tts)
- `failed` — any required stage failed and pipeline stopped

*(Narrative is non-blocking — pipeline doesn't fail overall if narrative fails)*

---

## Pipeline Dashboard View (what this enables)

```
novel_chapters with chapter_pipeline_runs joined:

│ Chapter │ Scrape │ Preprocess │ Characters │ Dialogue │ Narrative │ TTS     │ Overall  │
├─────────┼────────┼────────────┼────────────┼──────────┼───────────┼─────────┼──────────┤
│ Ch.  1  │   ✓    │     ✓      │     ✓      │    ✓     │     ✓     │    ✓    │ complete │
│ Ch.  2  │   ✓    │     ✓      │  ✗ failed  │  pending │  pending  │ pending │  failed  │
│         │        │            │ "API 429"  │          │           │         │          │
│ Ch.  3  │   ✓    │     ✓      │     ✓      │  ⟳ run  │  pending  │ pending │  running │
│ Ch.  4  │  pending│  pending  │  pending   │  pending │  pending  │ pending │  pending │
└─────────┴────────┴────────────┴────────────┴──────────┴───────────┴─────────┴──────────┘
```

A single query gives you the full picture for every chapter in a novel — no joins across multiple tables or digging through logs.

---

## New Service: `PipelineTrackingService`

Central service updated by every job and the pipeline orchestrator.

```php
// Create or retrieve the run record for a chapter
PipelineTrackingService::startRun(NovelChapter $chapter): ChapterPipelineRun

// Called at the start of each stage
PipelineTrackingService::startStage(NovelChapter $chapter, string $stage): void
// e.g. startStage($chapter, 'scrape')

// Called on stage success
PipelineTrackingService::completeStage(NovelChapter $chapter, string $stage): void

// Called on stage failure — stores the reason
PipelineTrackingService::failStage(NovelChapter $chapter, string $stage, string $error): void

// Called when a stage is skipped (already done — idempotent pipeline)
PipelineTrackingService::skipStage(NovelChapter $chapter, string $stage): void

// Reset for reprocessing — increments run_count, clears all statuses to pending
PipelineTrackingService::resetRun(NovelChapter $chapter, array $stages = []): void
// If $stages given, only reset those stages (partial reset)

// Recalculate and save overall_status
private function updateOverallStatus(ChapterPipelineRun $run): void
```

Stage name constants (on `ChapterPipelineRun` model):
```php
const STAGE_SCRAPE      = 'scrape';
const STAGE_PREPROCESS  = 'preprocess';
const STAGE_CHARACTERS  = 'characters';
const STAGE_DIALOGUE    = 'dialogue';
const STAGE_NARRATIVE   = 'narrative';
const STAGE_TTS         = 'tts';
```

---

## Files to Create / Modify

### New migration
- `database/migrations/*_create_chapter_pipeline_runs_table.php`

### New model
- `app/Models/ChapterPipelineRun.php`
  - `belongsTo(NovelChapter)`
  - Stage name constants
  - `hasFailed()`, `isComplete()`, `isRunning()` helpers
  - `durationFor(string $stage): ?int` — computed ms from started/completed timestamps

### New service
- `app/Services/PipelineTrackingService.php`
  - All `startStage` / `completeStage` / `failStage` / `skipStage` / `resetRun` methods
  - Uses `updateOrCreate` on `chapter_id` so upsert is safe from any caller
  - `updateOverallStatus()` recalculates and saves overall status after every change

### Modified: `app/Jobs/ProcessChapterPipeline.php`
Wire tracking into the orchestrator for scrape and preprocess stages:
```php
$tracking->startStage($chapter, 'scrape');
// ... scrape ...
$tracking->completeStage($chapter, 'scrape');
// or
$tracking->failStage($chapter, 'scrape', $e->getMessage());
```

### Modified: `app/Jobs/ProcessChapterCharactersWithLlm.php` (new job from restructure plan)
```php
$tracking->startStage($chapter, 'characters');
// ... run Pipeline 1 ...
$tracking->completeStage / failStage($chapter, 'characters', $error);
```

### Modified: `app/Jobs/ProcessChapterWithLlm.php`
```php
$tracking->startStage($chapter, 'dialogue');
// ...
$tracking->completeStage / failStage($chapter, 'dialogue', $error);
```

### Modified: `app/Jobs/ProcessChapterNarrativeWithLlm.php`
```php
$tracking->startStage($chapter, 'narrative');
// ...
$tracking->completeStage / failStage($chapter, 'narrative', $error);
```

### Modified: `app/Services/TtsService.php` (or `TtsController` callback handler)
Update `tts_started_at` on dispatch, `tts_completed_at` + `tts_error` on callback.

### Modified: `app/Http/Controllers/AdminPipelineController.php`
- Update `pipeline()` to include pipeline run record
- Update `pipelineSummary()` to include per-stage statuses from run record
- Update `resetLlmProcessing()` to call `PipelineTrackingService::resetRun()`

### New controller method / route
```
GET /api/admin/novels/{novel}/pipeline-runs
```
Returns one row per chapter with all stage statuses — the dashboard view shown above.

---

## Backwards Compatibility

- Existing `llm_processing_status` / `llm_narrative_status` columns on `novel_chapters` are **kept for now** — no breaking changes to existing code that reads them
- The `chapter_pipeline_runs` table is **additive** — existing integrations continue to work
- Eventually (future cleanup), the scattered columns on `novel_chapters` can be removed in favour of joining `chapter_pipeline_runs`

---

## On Reset / Reprocess

When `resetLlmProcessing()` is called:
1. `PipelineTrackingService::resetRun($chapter, ['characters', 'dialogue', 'narrative'])` — resets only LLM stages
2. `run_count++` — tracks how many attempts this chapter has had
3. Error fields cleared — fresh attempt
4. `overall_status` → `pending` or `running` depending on remaining stages

When full pipeline reset:
- `resetRun($chapter)` with no stage filter resets everything
- All stage statuses → `pending`, errors cleared, `run_count++`

---

## Phasing

1. **Migration + model** — `chapter_pipeline_runs` table + `ChapterPipelineRun` model
2. **`PipelineTrackingService`** — all tracking methods
3. **Wire into `ProcessChapterPipeline`** — scrape + preprocess stages tracked
4. **Wire into LLM jobs** — characters, dialogue, narrative stages tracked
5. **Wire into TTS** — tts stage tracked via dispatch + callback
6. **Controller updates** — `pipelineSummary`, `resetLlmProcessing`, new novel-level dashboard route
7. **Backfill** — for existing chapters, derive initial run status from current `novel_chapters` columns

---

*Written: 2026-02-28*
*Depends on: Pipeline Restructure Plan.md (implement first)*
