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

## PHASE 3 — Non-Human Character Support ✅ DONE

- ✅ DB migration: `race` varchar(100) default 'human' added to `novel_characters`
- ✅ `NovelCharacter` model: `race` in fillable; `CharacterController` validates + returns `race`
- ✅ LLM prompts: both `buildCharacterExtractionPrompt` and `buildChapterProcessingPrompt` include `race`; appearance guidance covers species-specific features; voice_suggestion still maps to human voice categories
- ✅ Avatar prompt: `CharacterImageService.buildPrompt()` is race-aware — humans get standard descriptor, humanoid races (elf, orc, dwarf, demon, undead, fairy) get species traits, beast/dragon characters get non-human form descriptions
- ✅ Avatar style upgraded from flat vector icon → anime cel-shading portrait with expressive eyes and richer personality trait → expression mapping (Phase 4 items folded in)
- ✅ Flutter: `NovelCharacter` model has `race` field, `isHuman` getter, `displayRace` helper; character list tiles and character sheet show race for non-humans

---

## PHASE 4 — Avatar Prompt Overhaul (Anime Style, More Personality) ✅ DONE (folded into Phase 3)

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

*Last updated: 2026-02-28 — Phases 1–4 complete*
