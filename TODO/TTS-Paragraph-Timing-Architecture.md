# TTS Paragraph Timing Architecture — `final_tts_paragraphs`

## Context

Analysis of live chapter TTS data (chapter 11010, 12:17 audio, 76 paragraphs) found paragraph highlighting is broken. Root cause: three different ID spaces are in play (original paragraph IDs, LLM segment IDs, TTS timing IDs), each used inconsistently across the pipeline. The fix is a new `final_tts_paragraphs` table — a denormalized, pipeline-final snapshot that holds everything the frontend needs: display content, speaker attribution, table metadata, and TTS timing. Flutter always pulls from this one table. No joins, no ID bridging.

---

## Bugs Being Fixed

| # | Bug | Impact |
|---|-----|--------|
| 1 | LLM segments use `LlmProcessedParagraph.id` but TTS timing uses `novel_chapter_paragraphs.id` | Wrong paragraph highlighted in Flutter |
| 2 | Paragraphs endpoint only expands attributed (dialogue) paragraphs — creates mixed ID schema | Inconsistent Flutter state |
| 3 | LLM re-run invalidates TTS audio with no detection mechanism | Audio speaks text that no longer exists |
| 4 | `paragraphs_total` set to segment count in multi-voice mode (90 not 76) | Wrong progress display |
| 5 | Read-time `paragraph_id` rewrite in `getChapterTts` is a no-op that obscures real bugs | Misleading code |

---

## New Table: `final_tts_paragraphs`

This is the **only** table the frontend ever reads for paragraph/timing data. It is rebuilt from scratch whenever any upstream pipeline stage completes.

```
novel_chapter_paragraphs  (raw source, scraper writes here)
llm_processed_paragraphs  (LLM dialogue segmentation, internal pipeline state)
         │
         ▼  [Finalization step — rebuilds completely]
final_tts_paragraphs      (frontend display + TTS timing, one row = one display unit)
         │
         ▼  [TTS generation writes timing back here]
chapter_tts               (audio file metadata only: audio_url, status, duration)
```

### Schema

```php
Schema::create('final_tts_paragraphs', function (Blueprint $table) {
    $table->id();
    $table->foreignId('chapter_id')
        ->constrained('novel_chapters')
        ->cascadeOnDelete();

    // Display order — sequential within chapter, recomputed on each finalization
    $table->unsignedInteger('order');

    // Display content
    $table->text('content');
    $table->boolean('is_dialogue')->default(false);
    $table->boolean('is_narration')->default(true);
    $table->boolean('is_table_row')->default(false);
    $table->boolean('is_section_header')->default(false);
    $table->unsignedInteger('table_group_id')->nullable();
    $table->jsonb('table_data')->nullable();

    // Speaker (null = narrator)
    $table->foreignId('speaker_id')
        ->nullable()
        ->constrained('novel_characters')
        ->nullOnDelete();

    // TTS timing (null until TTS complete)
    $table->foreignId('tts_chapter_tts_id')
        ->nullable()
        ->constrained('chapter_tts')
        ->nullOnDelete();
    $table->decimal('tts_start', 10, 3)->nullable();
    $table->decimal('tts_end', 10, 3)->nullable();
    $table->jsonb('tts_alignment')->nullable(); // [{word, start, end}, ...]

    // Traceability (not identity — IDs here are never exposed to frontend)
    $table->unsignedBigInteger('source_paragraph_id')->nullable();
    $table->unsignedBigInteger('source_llm_segment_id')->nullable();

    $table->timestamps();

    $table->unique(['chapter_id', 'order']);
    $table->index(['chapter_id']);
    $table->index(['tts_chapter_tts_id']);
});
```

**Key properties:**
- `final_tts_paragraphs.id` is the only ID the frontend ever sees
- `order` is 1-based, sequential, no gaps — the true display order
- No reference to `novel_chapter_paragraphs.id` or `llm_processed_paragraphs.id` is exposed to frontend
- The table is **replace-on-write** for a chapter: delete all rows for that chapter then re-insert
- Table rows (`is_table_row=true`) ARE included and dispatched to TTS

---

## Implementation Steps

### Step 1 — Migration: Create `final_tts_paragraphs`

**New file**: `chronicle-api/database/migrations/2026_03_XX_create_final_tts_paragraphs_table.php`

Schema as above.

### Step 2 — New Service: `FinalParagraphService`

**New file**: `chronicle-api/app/Services/FinalParagraphService.php`

Single public method: `finalize(NovelChapter $chapter): void`

Logic:
- Delete all existing final rows for this chapter
- Also delete any existing `chapter_tts` records (audio is now stale)
- If chapter `llm_processing_status = 'complete'`: load `LlmProcessedParagraph` rows (non-artifact, in order), grouped by original paragraph. For each original paragraph: if it has LLM segments use those, otherwise fall back to `processed_content` as a single narration row. Copy `is_table_row`, `is_section_header`, `table_group_id`, `table_data` from the original paragraph onto each row.
- If LLM not run: one row per `novel_chapter_paragraphs` row.
- Assign sequential `order` starting at 1 across all rows.
- Bulk insert with `FinalTtsParagraph::insert($rows)`.

### Step 3 — Trigger Finalization at Pipeline Completion Points

**Decisions:**
- Finalization only happens after LLM pipeline completes, not on scrape.
- Transition path: if `final_tts_paragraphs` is empty but `llm_processing_status = 'complete'`, lazily finalize on demand.

**Files to modify**:

1. `chronicle-api/app/Jobs/ProcessChapterWithLlm.php` — after LLM job succeeds, call `FinalParagraphService::finalize()`. This is the primary trigger.
2. `chronicle-api/app/Http/Controllers/NovelController.php::paragraphs` — if `final_tts_paragraphs` is empty AND `llm_processing_status = 'complete'`, call `finalize()` lazily. If LLM hasn't run yet, return `{"status": "pending_pipeline", "paragraphs": []}`.
3. `chronicle-api/app/Http/Controllers/TtsController.php::requestChapterTts` — if no `final_tts_paragraphs` exist yet but LLM is complete, call `finalize()` before dispatching TTS.

### Step 4 — Change TTS Dispatch to Send `final_tts_paragraphs` Rows

**File**: `chronicle-api/app/Services/TtsService.php::dispatchForChapter`

Fetch from `final_tts_paragraphs` instead of `llm_processed_paragraphs`. Each row becomes one `ChapterSegment` with `segment_id = final_tts_paragraphs.id`. `paragraphs_total` = `$finalParagraphs->count()` (rows, not LLM segments). Detect multi-voice by checking if any row has `speaker_id != null`.

Also update `TtsController::requestChapterTts` which has duplicate dispatch logic.

### Step 5 — Change TTS Callback to Write Timing to `final_tts_paragraphs`

**File**: `chronicle-api/app/Http/Controllers/TtsController.php::handleCompleteCallback`

The `paragraph_timings[*].segments[*].segment_id` values now equal `final_tts_paragraphs.id`. For each, update that row with `tts_chapter_tts_id`, `tts_start`, `tts_end`, `tts_alignment`. Stop writing `paragraph_timings` JSONB to `chapter_tts`.

### Step 6 — Paragraphs Endpoint: Return `final_tts_paragraphs` Directly

**File**: `chronicle-api/app/Http/Controllers/NovelController.php::paragraphs`

Replace entire conditional expansion logic with a single `FinalTtsParagraph::where('chapter_id',...)->orderBy('order')->get()`. Response includes `tts_start`, `tts_end`, `tts_alignment` on each row. No `is_llm_segment`, no `original_paragraph_id`, no joins.

### Step 7 — TTS GET Endpoint: Simplified

**File**: `chronicle-api/app/Http/Controllers/TtsController.php::getChapterTts`

Remove `enrichTimingsWithCharacters`, remove `paragraph_id` rewrite block (lines 243–258), remove `paragraph_timings` from response. Return only: `id`, `chapter_id`, `status`, `voice`, `speed`, `audio_url`, `duration_seconds`, `paragraphs_processed`, `paragraphs_total`, `progress_percent`, `error_message`.

### Step 8 — TTS Python Service: Fix `paragraphs_total`

**File**: `chronicle-tts/app/routers/tts.py::_process_chapter_multivoice`

In the "started" callback, `paragraphs_total` should be the number of unique `paragraph_id` values in the segments (i.e., `len({seg.paragraph_id for seg in segments})`). Now that `paragraph_id == segment_id` in the new model, this equals `len(segments)` anyway — but conceptually correct.

### Step 9 — Flutter: Read Timing from Paragraphs Response

**Files**: `lib/models/chapter_tts.dart`, `lib/models/paragraph_tts.dart`, `lib/services/global_tts_service.dart`

1. Paragraph model gets `ttsStart`, `ttsEnd`, `ttsAlignment` fields (nullable).
2. Remove `ParagraphTiming` array and all timing-matching logic from `GlobalTtsService`.
3. Build playback position lookup as a flat sorted list of `{id, ttsStart, ttsEnd}` from paragraphs. Binary search for the active row at current playback position.
4. TTS GET endpoint no longer returns `paragraph_timings` — Flutter stops parsing it.
5. Remove `originalParagraphId` and `isLlmSegment` from paragraph model.

### Step 10 — Deferred Cleanup

After all production chapters have been re-finalized and TTS regenerated:
- Drop `chapter_tts.paragraph_timings` JSONB column (migration)
- Optionally keep or drop `llm_processed_paragraphs` as internal pipeline state

---

## Files Changed Summary

| File | Change |
|---|---|
| `chronicle-api/database/migrations/2026_03_XX_create_final_tts_paragraphs_table.php` | **New** — create table |
| `chronicle-api/app/Models/FinalTtsParagraph.php` | **New** — Eloquent model |
| `chronicle-api/app/Services/FinalParagraphService.php` | **New** — `finalize()` logic |
| `chronicle-api/app/Jobs/ProcessChapterWithLlm.php` | Call `finalize()` on success |
| `chronicle-api/app/Services/TtsService.php` | Read from `final_tts_paragraphs`; fix `paragraphs_total` |
| `chronicle-api/app/Http/Controllers/NovelController.php` | Serve `final_tts_paragraphs` directly; lazy finalize |
| `chronicle-api/app/Http/Controllers/TtsController.php` | Write timing to rows; clean up `getChapterTts`; fix dispatch |
| `chronicle-tts/app/routers/tts.py` | Fix `paragraphs_total` in started callback |
| `chronicle-flutter/lib/models/chapter_tts.dart` | Add timing fields to paragraph model; drop old fields |
| `chronicle-flutter/lib/models/paragraph_tts.dart` | Simplify to flat timing per row |
| `chronicle-flutter/lib/services/global_tts_service.dart` | Flat position lookup; no paragraph-to-timing matching |

---

## Verification

1. **Finalization**: Run LLM pipeline on test chapter. Call `GET /api/chapters/{id}/paragraphs`. Confirm all rows come from `final_tts_paragraphs`, `id` values are that table's IDs, `tts_start` is null.

2. **TTS generation**: Request TTS. Confirm `paragraphs_total` = paragraph row count. After completion, call paragraphs endpoint — confirm `tts_start`/`tts_end` populated on each row.

3. **TTS endpoint**: Confirm `GET /api/chapters/{id}/tts` has no `paragraph_timings` in response.

4. **Staleness detection**: Re-run LLM on a chapter with complete TTS. Confirm `chapter_tts` deleted, `final_tts_paragraphs` rebuilt, timing null again.

5. **Flutter reader**: Start playback. Paragraph highlighting advances correctly with audio. No off-by-one. Dialogue segments highlight independently.

6. **Table rows**: Chapter with status tables — rows appear with `is_table_row=true`, display and TTS correctly.
