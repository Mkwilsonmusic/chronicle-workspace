# Pipeline Restructure — Focused LLM Stages

## Problem

The current Pass 1 LLM does too much at once: dialogue attribution, artifact detection, and new character detection — all with the full novel character list. At chapter 1000+ with 200+ characters, this causes confusion and poor attribution quality.

**Fix:** Split Pass 1 into two focused pipelines. Move all artifact detection to preprocessing. Give the dialogue LLM only the characters that actually appear in that chapter.

---

## New Pipeline Flow

```
[Raw Scraped Content]
        │
        ▼
┌──────────────────────────────────────────────┐
│  STAGE 1: Preprocessing (Rule-Based)         │
│                                              │
│  • Unicode decoding                          │
│  • Scene break / author note exclusion       │
│  • CJK removal                               │
│  • Bracket cleaning                          │
│  • Table detection                           │
│  • Whitespace normalization                  │
│                                              │
│  ► ALL artifact detection lives here         │
│    (improve rules; AI cleanup = future pass) │
└────────────────────┬─────────────────────────┘
                     │ clean paragraphs (excluded=false)
                     ▼
┌──────────────────────────────────────────────┐
│  PIPELINE 1: Character Association           │
│  status: llm_character_status                │
│                                              │
│  Input:  preprocessed paragraphs             │
│          ALL known novel characters          │
│          (full list OK — just scanning)      │
│                                              │
│  Tasks:                                      │
│  • Which known characters appear here?       │
│  • Any genuinely new characters?             │
│  • Create new characters (with voice)        │
│  • Write chapter_character_appearances rows  │
│                                              │
│  Output: chapter_character_appearances table │
│          (typically ~5–20 chars per chapter) │
└────────────────────┬─────────────────────────┘
                     │ characters_in_chapter[]
                     ▼
┌──────────────────────────────────────────────┐
│  PIPELINE 2: Dialogue Attribution            │
│  status: llm_processing_status               │
│                                              │
│  Input:  preprocessed paragraphs             │
│          ONLY characters from Pipeline 1     │
│          (5–20 chars, not 200+)              │
│                                              │
│  Tasks:                                      │
│  • Split mixed dialogue/narration paragraphs │
│  • Attribute each dialogue to a speaker      │
│    from the chapter character list only      │
│                                              │
│  NO artifact detection (preprocessing owns) │
│  NO new character detection (Pipeline 1 owns)│
│  Simple focused prompt → fewer LLM errors   │
│                                              │
│  Output: llm_processed_paragraphs rows       │
└────────────┬──────────────┬──────────────────┘
             │              │
             │              └──────► [TTS Generation]
             ▼                       (multi-voice, unblocked)
┌──────────────────────────────────────────────┐
│  PIPELINE 3: Narrative Analysis              │
│  status: llm_narrative_status                │
│                                              │
│  Input:  speaker-attributed text             │
│          characters in chapter (from P1)     │
│                                              │
│  Tasks:                                      │
│  • Chapter summary                           │
│  • Key events                                │
│  • Relationship changes                      │
│                                              │
│  Output: chapter_summaries                   │
│          character_relations                 │
└──────────────────────────────────────────────┘
```

**Auto-dispatch chain:**
`ProcessChapterPipeline` (scrape + preprocess) → dispatches `ProcessChapterCharactersWithLlm`
→ success → dispatches `ProcessChapterWithLlm` (dialogue)
→ success → dispatches `ProcessChapterNarrativeWithLlm` + triggers TTS

---

## Why This Scales

| Novel size     | Old Pass 1 character list | New Pipeline 2 character list |
|----------------|--------------------------|-------------------------------|
| Ch 1–50        | ~20 characters            | ~5–15 characters              |
| Ch 100–200     | ~60 characters            | ~5–15 characters              |
| Ch 500–1000    | 200+ characters           | ~5–15 characters              |

Pipeline 1 handles the wide scan efficiently (just finding presence, not attributing lines). Pipeline 2 always gets a small, chapter-scoped list regardless of novel size.

---

## What Changes

### Schema

**New table: `chapter_character_appearances`**
- `chapter_id` FK → novel_chapters (cascade delete)
- `character_id` FK → novel_characters (cascade delete)
- `is_new_appearance` bool — true if first time this character appears in the novel
- unique on `[chapter_id, character_id]`

**New columns on `novel_chapters`:**
- `llm_character_status` varchar(50) default 'pending'
- `llm_character_processed_at` timestamp nullable

### New Files

| File | Purpose |
|------|---------|
| `app/Jobs/ProcessChapterCharactersWithLlm.php` | Pipeline 1 job; on success auto-dispatches Pipeline 2 |
| `app/Models/ChapterCharacterAppearance.php` | Pivot model for chapter ↔ character |
| `database/migrations/*_create_chapter_character_appearances_table.php` | Schema |
| `database/migrations/*_add_llm_character_status_to_novel_chapters.php` | Schema |

### Modified: `LlmProcessingService.php`

1. **Add `processChapterCharacters(NovelChapter $chapter)`** — Pipeline 1 logic
   - Load all novel characters + chapter paragraphs
   - Call new `buildCharacterPresencePrompt()`
   - Create `ChapterCharacterAppearance` rows
   - Create any genuinely new characters (reuse existing voice-assignment logic)
   - Set `llm_character_status = complete`

2. **Add `buildCharacterPresencePrompt($paragraphs, $characters)`**
   - Prompt: scan for character presence, return `{ "characters_present": [...], "new_characters": [...] }`
   - Simpler than current Pass 1 — just detection, no attribution

3. **Refactor `processChapter()` → `processChapterDialogue()`** — Pipeline 2 (simplified)
   - Load characters from `chapter_character_appearances` for this chapter only
   - Remove `new_characters` handling
   - Remove `is_artifact` / `artifact_reason` from LLM prompt (keep DB columns; preprocessing owns this)
   - Simpler, shorter prompt with small character list

4. **Modify `processChapterNarrative()`** — Pipeline 3
   - Load characters from `chapter_character_appearances` instead of all novel characters

### Modified: `ProcessChapterPipeline.php`
- Stage 3: dispatch `ProcessChapterCharactersWithLlm` (not `ProcessChapterWithLlm` directly)

### Modified: `ProcessChapterWithLlm.php`
- Call `processChapterDialogue()` instead of `processChapter()`

### Modified: `LlmProcessingController.php`
- Add `processChapterCharacters(chapter)` endpoint for manual retrigger
- Update `novelStatus()` to show `llm_character_status`

### Modified: `AdminPipelineController.php`
- Update pipeline visualization to show 3 LLM stages + character appearances

---

## New Routes

```
POST /api/admin/chapters/{chapter}/llm-characters        → processChapterCharacters
GET  /api/admin/chapters/{chapter}/character-appearances → list characters in chapter
```

---

## What Does NOT Change

- `initializeNovelCharacters()` — bootstrap step for new novels, still useful, unchanged
- All preprocessing rules — no changes (improve separately)
- `LlmProcessedParagraph` schema — unchanged
- `ChapterSummary`, `CharacterRelation` models — unchanged
- TTS generation — unchanged; still triggered after Pipeline 2 complete
- Voice assignment logic — reused in Pipeline 1

---

## Phasing

1. **Migrations** — create new table + columns
2. **Pipeline 1** — new job, new service method, new prompt
3. **Pipeline 2 simplification** — refactor existing dialogue pass
4. **Pipeline 3 minor update** — use chapter character list
5. **`ProcessChapterPipeline` update** — wire the new dispatch chain
6. **Controller / route additions** — expose new endpoints
7. **`initializeNovelCharacters` update** — populate `chapter_character_appearances` as a side-effect of bootstrap run

---

*Written: 2026-02-28*
