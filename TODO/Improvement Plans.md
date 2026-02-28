# Chronicle — Improvement Plans

---

## PHASE 1 — Post-LLM Chapters with Enhanced Highlighting ✅ DONE

- ✅ Dual filter: API `chapters()` now requires both `tts_status=complete` AND `llm_processing_status=complete`. Flutter novel detail passes both filters.
- ✅ Multi-voice TTS filter fixed: `whereHas('tts')` now uses `whereIn('voice', [$voice, 'multi'])` so multi-voice chapters appear in list.
- ✅ Character token pipeline confirmed correct end-to-end (API injects `character_id` via `enrichTimingsWithCharacters()`; Flutter parses in `SegmentTiming.fromJson`).
- ✅ Paragraph timing lookup switched to ID-based (`getTimingForParagraphId`) with index fallback.
- ✅ Admin route hardening: all admin ops moved to `/admin/` prefix + `EnsureUserIsAdmin` in API; chronicle-admin `api.js` updated to match; `requestChapterTts` in Flutter fixed to `/admin/` route.

---

## PHASE 2 — Character Improvements ✅ DONE

### 2a — Spoiler-Safe Character Detail View (Flutter only) ✅ DONE
- ✅ `CharacterListScreen` filters characters by `firstAppearanceChapter.chapter_index ≤ bookmarkChapterIndex` — characters not yet encountered are hidden.
- ✅ `novel_detail_screen` passes `bookmarkChapterId` + `bookmarkChapterIndex` (looked up from loaded `_chapters`) when navigating to `/characters`.
- ✅ `CharacterSheet.show` called with `currentChapterId: bookmarkChapterId` so events are gated to the reader's current position.

### 2b — Two-Pass LLM Processing + Character Relationships ✅ DONE (API/backend)

**Architecture decision:** LLM processing split into two focused passes:
- **Pass 1 (dialogue)** — `llm_processing_status`: speaker attribution, paragraph splitting, new character detection. Unblocks TTS. Fast, runs first.
- **Pass 2 (narrative)** — `llm_narrative_status`: events, chapter summary, relationship changes. Non-blocking, auto-dispatched after Pass 1. Gets speaker-attributed text as input, making relationship detection significantly more accurate.

**Completed:**
- ✅ Migration: `llm_narrative_status` + `llm_narrative_processed_at` on `novel_chapters`
- ✅ Migration: `character_relations` table (character_id, related_character_id, relation_type, started_chapter_id, ended_chapter_id, notes)
- ✅ Model: `CharacterRelation`
- ✅ `LlmProcessingService::processChapterNarrative()` — Pass 2 method using speaker-attributed text from Pass 1
- ✅ Pass 1 auto-dispatches `ProcessChapterNarrativeWithLlm` job on success
- ✅ Job: `ProcessChapterNarrativeWithLlm`
- ✅ Controller: `processChapterNarrative()` and `processChaptersNarrative()` for manual re-trigger
- ✅ `novelStatus()` now shows both dialogue and narrative status per chapter
- ✅ Routes: `POST /admin/chapters/{id}/llm-narrative`, `POST /admin/novels/{id}/llm-narrative`
- ✅ Route: `GET /characters/{id}/relations?up_to_chapter={id}` (spoiler-gated, public auth)
- ✅ Admin api.js: `processChapterNarrative()`, `processChaptersNarrative()`

**Still needed (Flutter):**
- Character sheet relations tab — call `GET /characters/{id}/relations` and display (deferred to Phase 5 polish)

---

## PHASE 3 — Non-Human Character Support (Animals, Mythical Creatures, Beasts)

Web novels are full of non-human characters — loyal spirit beasts, monster companions, elves, orcs, dragons, etc. The current system assumes all characters are human, which breaks avatar generation and limits character depth.

### 3a — DB: Add `race` column to `novel_characters`

- **New column**: `race` (varchar 100, nullable, default `human`)
- Examples: `human`, `elf`, `orc`, `dwarf`, `dragon`, `wolf`, `beast`, `spirit_beast`, `demon`, `undead`, `fairy`, `automaton`
- **No change needed to `gender`** — animals and mythical creatures can still be male/female for TTS voice assignment. Gender stays as-is.
- Migration: `ALTER TABLE novel_characters ADD COLUMN race VARCHAR(100) DEFAULT 'human'`

### 3b — LLM: Update character extraction prompt

Update `buildCharacterExtractionPrompt` in `LlmProcessingService` to:
- Add `race` to the extraction schema (e.g. `"race": "spirit_beast"`)
- Update appearance guidance to note species-specific features (e.g. pointed ears for elves, fur/claws for beasts)
- Updated voice_suggestion should still map to the human voice categories — a male wolf beast gets `mature_male`, etc.
- Example addition to the extraction system prompt:
  ```
  - race: The character's species/race. Use "human" as default. For non-humans use descriptive terms like:
    "elf", "orc", "dwarf", "dragon", "spirit_beast", "demon_beast", "wolf_beast", "fairy", "undead", etc.
    Be specific where the text allows (e.g. "black-scaled dragon" → race: "dragon").
  ```

Also update `buildChapterProcessingPrompt` so `new_characters` includes `race`.

### 3c — Avatar prompt: Handle non-human races

Update `buildPrompt` in `CharacterImageService`:
- Check `character->race` — if not human, inject the race prominently into the prompt.
- For beasts/animals: describe the animal form, not a human.
- For humanoid races (elf, orc, dwarf): describe human-like figure with species traits.
- Example:
  - Human: `"young adult male, lean athletic build, short black hair"` → standard anime portrait
  - Elf: `"young adult male elf, long pointed ears, ethereal pale skin, silver hair"`
  - Spirit Beast / Wolf: `"anthropomorphic wolf spirit beast, silver fur, piercing amber eyes, proud bearing"` — head-and-shoulders portrait
  - Dragon: `"dragon character icon, reptilian head and neck, scales, fierce golden eyes, smoke at nostrils"`

---

## PHASE 4 — Avatar Prompt Overhaul (Anime Style, More Personality)

**Problem with current prompt:**
> `"Minimalist character icon: {descriptor}. Style: flat vector icon, single solid color silhouette with simple facial features. Pure white background, centered head only, no neck or shoulders."`

This produces flat, generic, expressionless icons with no personality — not what a reader invested in web novel characters wants to see.

**Goals:**
- Style that resonates with anime/light novel/manhwa readers
- Characters should feel emotionally distinct and recognisable
- Expressive faces — eyes especially (anime's most characterful feature)
- Consistent icon format (head + shoulders crop, light background) — size is already perfect
- Still works as a small circular token in the reader

### Suggested new prompt template:

```
Anime-style character portrait icon: {descriptor}, {expression}.
Style: clean anime illustration with expressive eyes, soft cel-shading,
subtle colour depth and dynamic hair. Head and shoulders crop,
very light neutral background, centered composition.
The character should feel emotionally resonant and immediately distinct —
not generic. Sharp confident linework, polished manga aesthetic.
```

**Expression mapping** — upgrade `selectTraitsForPrompt`:
| Trait | Current | Suggested upgrade |
|-------|---------|-------------------|
| fierce | `intense gaze` | `sharp piercing eyes, battle-hardened look, barely-contained intensity` |
| mysterious | `enigmatic expression` | `half-lidded knowing eyes, slight enigmatic smile, aura of hidden depths` |
| arrogant | `haughty expression` | `chin raised, cold dismissive eyes, the look of someone who's never lost` |
| cunning | `sly smile` | `sharp intelligent eyes, a corner of the mouth curled in a calculated smile` |
| noble | `dignified expression` | `composed regal bearing, calm authoritative gaze, dignified set of the jaw` |
| brave | `determined expression` | `unwavering eyes full of conviction, set jaw, the look of someone who won't back down` |
| villain | `menacing expression` | `cold predatory eyes, cruel smile, an aura of dangerous calm` |
| gentle | `gentle expression` | `warm soft eyes, kind slight smile, an approachable calming presence` |
| cheerful | `smiling` | `bright sparkling eyes, wide genuine smile, radiating infectious energy` |
| mysterious | `enigmatic expression` | `distant half-lidded eyes, faint smile that hides more than it reveals` |

**Additional style anchors to include based on novel genre:**
- Cultivation/Xianxia: `"wuxia aesthetic, flowing robes hint at shoulders, spiritual aura"`
- Fantasy/RPG: `"fantasy RPG character portrait, equipment glimpsed at collar"`
- Sci-fi/Virtual: `"sleek futuristic aesthetic, technological elements at collar"`

**Shoulder crop note:** The user confirmed they like the shoulder-height crop.
Change `"centered head only, no neck or shoulders"` → `"head and shoulders portrait, cropped at chest"`.

---

## PHASE 5 — Reader Experience Polish

*Lower priority items to tackle after Phases 1-4.*

- **Chapter filter also require `llm_processing_status=complete`** — ensure both LLM and TTS must be done before a chapter appears in the list.
- **Characters hidden until first appearance** — `CharacterListScreen` and inline tokens should not reveal characters the reader hasn't encountered yet (gated by `first_appearance_chapter_id` vs user's `last_tts_chapter_id` bookmark).
- **Relation spoiler gating** — `GET /characters/{character}/events` already uses chapter gating; same logic needed for relations endpoint.
- **Bookmark auto-update** — ensure `last_tts_chapter_id` on `novel_user` is reliably updated whenever the user completes a chapter.

---

*Last updated: 2026-02-28 — Phase 1 complete, Phase 2 complete*
