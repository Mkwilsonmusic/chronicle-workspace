# Flutter Frontend Plan: Character Display & LLM Output Integration

## Overview

Show speaking characters inline with text during playback, with tappable avatars that display character info and story events.

**User Preferences (from discussion):**
- Character tokens appear **inline with text**
- AI image generation via **OpenAI DALL-E**
- Character popup shows **basic info + recent events**

---

## Current State

### Backend (Already Exists)
- `NovelCharacter` model: name, aliases, gender, personality_traits, description, tts_voice, is_protagonist
- `LlmProcessedParagraph` model: speaker_id, is_dialogue, content, split_order
- `SegmentTiming` in TTS: includes voice, is_dialogue, segment_id
- API endpoints: `/api/admin/novels/{id}/characters`, `/api/admin/characters/{id}`
- Chapter summaries: key_events, characters_present

### Flutter (Current)
- `HighlightedParagraph` widget: word-level highlighting during playback
- `SegmentTiming` model: has voice/is_dialogue but not currently used for display
- `GlobalTtsService`: tracks current segment during multi-voice playback
- No character display infrastructure

---

## Implementation Plan

### Phase 1: Character Data Integration

#### 1.1 Add Character Models to Flutter

**New file: `lib/models/character.dart`**
```dart
class Character {
  final int id;
  final int novelId;
  final String name;
  final List<String> aliases;
  final String? gender;
  final List<String> personalityTraits;
  final String? description;
  final String? ttsVoice;
  final bool isProtagonist;
  final String? avatarUrl;  // DALL-E generated image
  final Map<String, dynamic> profileData;
}
```

#### 1.2 Add Character API Methods

**Update: `lib/services/api_service.dart`**
```dart
// Get all characters for a novel
Future<List<Character>> getNovelCharacters(int novelId);

// Get single character with events
Future<CharacterDetail> getCharacter(int characterId, {int? upToChapterId});

// Get characters present in a chapter
Future<List<Character>> getChapterCharacters(int chapterId);
```

#### 1.3 Backend: Add Character Events Endpoint

**New endpoint needed: `GET /api/characters/{id}/events?up_to_chapter={id}`**
Returns:
- Character basic info
- Events involving this character (from chapter_summaries.key_events)
- Filtered to chapters <= up_to_chapter (no spoilers)

---

### Phase 2: Character Avatar Generation

#### 2.1 Cost Analysis & Model Selection

**DALL-E 3 Pricing:**
- Minimum size: 1024x1024 (no smaller options)
- Standard: $0.04/image
- HD: $0.08/image

**Recommended: GPT Image 1 Mini (Low quality)**
- Cost: **$0.005-0.006/image** (~8x cheaper)
- Good enough for 24px tokens - we'll downscale anyway
- Supports smaller output sizes

**Per-novel cost estimate:**
- 10 characters × $0.006 = **$0.06 per novel**
- 50 characters × $0.006 = **$0.30 per novel**

#### 2.2 Character Data for Prompt

From `NovelCharacter` model (extracted by LLM):
```
name: "Zhao Yueru"
gender: "female"
personality_traits: ["elegant", "cunning", "cold"]
description: "A young woman with sharp features and an aloof demeanor"
voice_category: "young_female" → implies age
is_protagonist: false
```

#### 2.3 Prompt Design for Small Tokens

**Requirements:**
- Extremely simple - must read well at 24-32px
- Pure white background (or transparent)
- Head/face only, minimal detail
- Consistent style across all characters

**Prompt Template:**
```
Minimalist character icon: [gender] [age_from_voice], [1-2 key traits].
Style: flat vector icon, single solid color silhouette with simple facial features.
Pure white background, centered head only, no neck or shoulders.
Like a user avatar icon or game character select portrait.
```

**Example prompts:**
```
"Minimalist character icon: young woman, elegant and cold.
Style: flat vector icon, single solid color silhouette with simple facial features.
Pure white background, centered head only."

"Minimalist character icon: mature man, authoritative and serious.
Style: flat vector icon, single solid color silhouette with simple facial features.
Pure white background, centered head only."
```

**Why this style:**
- Flat vector = clean at small sizes
- Silhouette = no complex details to lose
- Single color = distinct per character
- No background = transparent effect

#### 2.4 Backend: CharacterImageService

**New file: `app/Services/CharacterImageService.php`**
```php
class CharacterImageService
{
    private string $model = 'gpt-image-1-mini';  // Cheaper
    private string $quality = 'low';             // $0.006/image
    private string $size = '1024x1024';          // Minimum for DALL-E 3

    public function generateAvatar(NovelCharacter $character): array
    {
        $prompt = $this->buildPrompt($character);
        $response = $this->callImageApi($prompt);

        // Download and store in S3 (or local)
        $imageUrl = $this->storeImage($response['url'], $character);

        $character->update([
            'avatar_url' => $imageUrl,
            'avatar_generated_at' => now(),
        ]);

        return ['url' => $imageUrl, 'prompt' => $prompt];
    }

    private function buildPrompt(NovelCharacter $character): string
    {
        $gender = $character->gender ?? 'person';
        $age = $this->ageFromVoiceCategory($character->voice_category);
        $traits = implode(', ', array_slice($character->personality_traits ?? [], 0, 2));

        return "Minimalist character icon: {$age} {$gender}" .
               ($traits ? ", {$traits}" : "") . ".\n" .
               "Style: flat vector icon, single solid color silhouette with simple facial features.\n" .
               "Pure white background, centered head only, no neck or shoulders.";
    }
}
```

#### 2.5 Backend: Avatar Endpoints

```
POST /api/admin/characters/{id}/generate-avatar
  → Triggers generation, returns {url, prompt_used, cost}

GET /api/admin/characters/{id}/avatar
  → Returns image URL (or 404 if not generated)

POST /api/admin/novels/{id}/generate-all-avatars
  → Batch generate for all characters (with progress)
```

#### 2.6 Admin UI: Character Avatar Management

**New Flutter screen: `lib/screens/admin/character_management_screen.dart`**

Accessible from novel detail (admin only) or dedicated admin section.

**Features:**
- List all characters for a novel
- Show current avatar (or placeholder)
- "Generate Avatar" button per character
- "Generate All" batch action
- Preview prompt before generating
- Regenerate option

**UI mockup:**
```
┌─────────────────────────────────────────┐
│ Characters - "Reincarnation of..."      │
├─────────────────────────────────────────┤
│ ┌──────┐  Zhao Yueru                    │
│ │ [img]│  young_female • protagonist    │
│ │      │  elegant, cunning              │
│ └──────┘  [Generate] [Preview Prompt]   │
├─────────────────────────────────────────┤
│ ┌──────┐  Gentle Snow                   │
│ │  ?   │  mature_female                 │
│ │      │  cold, taciturn                │
│ └──────┘  [Generate] [Preview Prompt]   │
├─────────────────────────────────────────┤
│         [Generate All Avatars]          │
│         Est. cost: $0.06 (10 chars)     │
└─────────────────────────────────────────┘
```

**Preview dialog:**
Shows the exact prompt that will be sent, allows editing before generation.

#### 2.7 Flutter: Avatar Caching

**Update: `lib/services/api_service.dart`**
- Cache character avatars locally (base64 or file)
- Fallback to placeholder based on gender/voice

**Fallback avatars (when not generated):**
- Colored circle with first letter of name
- Color based on character's assigned TTS voice
- Simple gender icon as fallback

---

### Phase 3: Inline Character Display

#### 3.1 Character Token Widget

**New file: `lib/widgets/character_token.dart`**
```dart
class CharacterToken extends StatelessWidget {
  final Character character;
  final bool isActive;  // Currently speaking
  final VoidCallback onTap;

  // Displays:
  // - Small circular avatar (24px)
  // - Subtle glow/border when active
  // - Tap to show character sheet
}
```

#### 3.2 Update HighlightedParagraph for Segments

**Modify: `lib/widgets/highlighted_paragraph.dart`**

Currently renders plain text with word highlighting. Need to:
1. Accept segment data (which parts are dialogue, who speaks)
2. Render character token at start of dialogue segments
3. Maintain word-level highlighting within segments

**New props:**
```dart
final List<SegmentDisplay>? segments;  // Optional multi-voice data
final Character? currentSpeaker;
final VoidCallback? onCharacterTap;
```

**Rendering logic:**
```
For each segment in paragraph:
  - If is_dialogue and has speaker:
    - Show CharacterToken at segment start
    - Apply dialogue styling (optional: slight indent, quote marks)
  - Render text with existing highlight animation
```

#### 3.3 Update ChapterReaderScreen

**Modify: `lib/screens/chapter_reader_screen.dart`**

1. Load chapter characters on init
2. Map segment voice → character (via tts_voice field)
3. Pass segment data to HighlightedParagraph
4. Handle character token taps → show bottom sheet

---

### Phase 4: Character Info Sheet

#### 4.1 Character Bottom Sheet

**New file: `lib/widgets/character_sheet.dart`**
```dart
class CharacterSheet extends StatelessWidget {
  final Character character;
  final List<CharacterEvent> recentEvents;
  final int currentChapterId;  // For spoiler filtering

  // Shows:
  // - Large avatar
  // - Name + aliases
  // - Description
  // - Personality traits as chips
  // - "Story so far" section with key events
}
```

#### 4.2 Character Event Model

**New file: `lib/models/character_event.dart`**
```dart
class CharacterEvent {
  final int chapterId;
  final String chapterTitle;
  final String eventDescription;
  final String spoilerLevel;  // mild, moderate, heavy
}
```

#### 4.3 Events Display

- Show events from chapters user has completed or is currently reading
- Filter by spoiler_level preference (future: user setting)
- Group by chapter for easy scanning
- Tap event → option to navigate to that chapter

---

### Phase 5: State Management & Performance

#### 5.1 Character Provider

**New file: `lib/services/character_service.dart`**
```dart
class CharacterService extends ChangeNotifier {
  final Map<int, Character> _novelCharacters = {};  // novelId → characters
  final Map<int, Uint8List> _avatarCache = {};  // characterId → image bytes

  Future<List<Character>> getCharactersForNovel(int novelId);
  Character? getCharacterByVoice(int novelId, String voice);
  Future<Uint8List?> getAvatar(int characterId);
}
```

#### 5.2 Preloading Strategy

- Load novel characters when entering NovelDetailScreen
- Preload avatars for protagonist + frequently appearing characters
- Lazy-load other avatars on first appearance

#### 5.3 Segment-to-Character Mapping

The `SegmentTiming` model already has `voice` field. Need to:
1. Build voice → character lookup per novel
2. Cache this mapping in CharacterService
3. Look up during paragraph rendering

---

## API Changes Summary

### New Backend Endpoints
| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/admin/characters/{id}/generate-avatar` | Trigger image generation |
| POST | `/api/admin/novels/{id}/generate-all-avatars` | Batch generate all character avatars |
| GET | `/api/admin/characters/{id}/avatar` | Get avatar URL |
| GET | `/api/characters/{id}/events` | Get character events (spoiler-filtered) |
| GET | `/api/chapters/{id}/characters` | Characters appearing in chapter |

### New Backend Columns
| Table | Column | Type | Purpose |
|-------|--------|------|---------|
| `novel_characters` | `avatar_url` | text nullable | Generated image URL (S3) |
| `novel_characters` | `avatar_generated_at` | timestamp nullable | For cache invalidation |
| `novel_characters` | `avatar_prompt` | text nullable | Store prompt for regeneration |

### Flutter API Methods
```dart
getNovelCharacters(int novelId)
getCharacterEvents(int characterId, {int? upToChapterId})
getChapterCharacters(int chapterId)
generateCharacterAvatar(int characterId)
```

---

## File Changes Summary

### New Flutter Files
```
lib/
  models/
    character.dart                      # Character model
    character_event.dart                # Event model
  services/
    character_service.dart              # Character state + caching
  widgets/
    character_token.dart                # Inline avatar widget
    character_sheet.dart                # Bottom sheet detail view
  screens/
    admin/
      character_management_screen.dart  # Admin: manage character avatars
```

### Modified Flutter Files
```
lib/
  services/
    api_service.dart                    # Add character endpoints
  widgets/
    highlighted_paragraph.dart          # Add segment rendering
  screens/
    chapter_reader_screen.dart          # Integrate character display
    novel_detail_screen.dart            # Add link to admin character management
  main.dart                             # Add CharacterService provider, routes
```

### New Backend Files
```
app/
  Services/
    CharacterImageService.php           # OpenAI image generation
database/
  migrations/
    xxxx_add_avatar_to_characters.php   # Avatar columns
```

### Modified Backend Files
```
app/
  Http/Controllers/
    CharacterController.php             # Add avatar endpoints
routes/
  api.php                               # New routes
config/
  services.php                          # Add OpenAI image config
```

---

## Implementation Order

### Phase A: Admin Avatar Generation (Start Here)
1. **Backend: Migration** - Add avatar columns to novel_characters
2. **Backend: CharacterImageService** - OpenAI image generation service
3. **Backend: Avatar endpoints** - generate-avatar, get avatar, batch generate
4. **Flutter: Admin character management screen** - trigger generation, view results
5. **Test & iterate on prompts** - refine style for small tokens

### Phase B: Reader Integration
6. **Flutter: Character models & API** (data layer)
7. **Flutter: CharacterService** (state management + caching)
8. **Flutter: CharacterToken widget** (inline display)
9. **Flutter: Update HighlightedParagraph** (segment rendering)
10. **Flutter: CharacterSheet** (detail view)

### Phase C: Events & Polish
11. **Backend: Events endpoint** (story-so-far feature)
12. **Integration testing & polish**

---

## Open Questions

1. **Model choice**: GPT Image 1 Mini ($0.006) vs DALL-E 3 ($0.04)?
   - Recommend Mini for cost, but quality may vary
   - Could test both on same character to compare

2. **Avatar regeneration**: Should admin be able to regenerate if unhappy?
   - Yes, with "Regenerate" button that shows cost warning
   - Store previous prompts for iteration

3. **Prompt refinement**: May need to iterate on prompt style
   - Start with flat vector silhouette approach
   - Could try: anime style, emoji style, geometric shapes
   - Admin preview lets us test before committing

4. **Storage**: Where to store generated images?
   - Option A: S3 (existing infrastructure)
   - Option B: Base64 in database (simpler, works offline)
   - Recommend S3 for consistency with TTS audio

5. **Performance**: With many characters speaking in quick succession:
   - Pre-render character tokens for visible paragraphs
   - Use `RepaintBoundary` around tokens
   - Cache all avatars when chapter loads

6. **Non-LLM chapters**: For chapters without LLM processing:
   - Show nothing (current behavior) - recommended
   - Or show narrator icon for all paragraphs

---

## Future Enhancements (Out of Scope)

- Character relationship graph
- Character voice preview (play TTS sample)
- User-editable character descriptions
- Character appearance history (when did they show up)
- Spoiler settings (hide events from unread chapters)
