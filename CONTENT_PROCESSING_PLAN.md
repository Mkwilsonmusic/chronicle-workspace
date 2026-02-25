# Chronicle Content Processing Pipeline — Master Plan

> **Purpose:** This document captures the complete vision and implementation plan for Chronicle's multi-stage content processing system, including raw content preservation, rule-based preprocessing, LLM-powered enhancements, and multi-voice TTS.

## Overview

The content processing pipeline transforms scraped novel content through multiple stages:

```
SCRAPE → RAW CONTENT → RULE PROCESSING → LLM PROCESSING → TTS GENERATION
  │           │              │                 │               │
  │           │              │                 │               └── Multi-voice audio
  │           │              │                 └── Characters, dialogue, summaries
  │           │              └── Tables, artifacts, cleaning
  │           └── Immutable original text
  └── NovelBin scraper
```

## Architecture

### Three-Layer Content Model

| Layer | Column | Mutability | Purpose |
|-------|--------|------------|---------|
| **Raw** | `raw_content` | Immutable | Original scraped text, never modified |
| **Processed** | `processed_content` | Mutable | Rule-based cleaning, table detection |
| **LLM** | `llm_processed_paragraphs` | Mutable | Dialogue attribution, character detection |

### Database Schema

**Core Tables:**
- `novel_chapter_paragraphs` — Raw + processed content, table detection
- `novel_characters` — Character profiles with voice assignments
- `llm_processed_paragraphs` — Speaker attribution, dialogue splitting
- `chapter_summaries` — Key events with spoiler levels

**Supporting Tables:**
- `parse_rules` — Versioned preprocessing rules
- `parse_rule_applications` — Audit trail of rule effects

### Processing Stages

**Stage 1: Scraping**
- Source: NovelBin (novelbin.me/.com)
- Output: `raw_content` (immutable), `processed_content` (copy)
- Trigger: Manual via `/api/chapters/{id}/scrape-content`

**Stage 2: Rule-Based Processing**
- Input: `processed_content`
- Operations: Unicode decode, HTML entities, CJK removal, artifact exclusion
- Table Detection: Status screens, stat blocks, equipment lists
- Output: Modified `processed_content`, `excluded` flag, table metadata

**Stage 3: LLM Processing (GPT-4o-mini)**
- Input: `processed_content`, character profiles
- Operations:
  - Character identification (novel initialization)
  - Dialogue attribution (who is speaking)
  - Paragraph splitting (separate dialogue from narration)
  - Artifact detection (translator notes, etc.)
  - Table validation (confirm status screens)
  - Chapter summarization with key events
- Output: `llm_processed_paragraphs`, `chapter_summaries`

**Stage 4: TTS Generation**
- Input: `llm_processed_paragraphs` (or `processed_content` fallback)
- Multi-voice: Different Kokoro voice per character
- Audio: Concatenated segments, AAC/M4A format, S3 storage

## LLM Integration

### Model Selection
- **Primary:** GPT-4o-mini ($0.15/1M input, $0.60/1M output)
- **Estimated cost:** ~$0.001 per chapter, ~$1 per 1000 chapters

### Prompts

**Novel Initialization (first 5 chapters):**
```json
{
  "characters": [{
    "name": "Ethan",
    "aliases": ["The Hero"],
    "gender": "male",
    "personality": ["determined", "cautious"],
    "voice_suggestion": "young_male"
  }],
  "protagonist": "Ethan",
  "narration_style": "third_person"
}
```

**Per-Chapter Processing:**
```json
{
  "paragraphs": [{
    "index": 0,
    "speaker": "Narrator",
    "is_dialogue": false,
    "split_points": [],
    "is_artifact": false,
    "table_valid": null
  }],
  "new_characters": [],
  "events": [{"description": "...", "significance": "minor"}],
  "summary": "Chapter summary"
}
```

### Voice Mapping

| Voice Category | Kokoro Voice | Grade | Use Case |
|----------------|--------------|-------|----------|
| `young_female` | `af_bella` | A- | Young female characters |
| `mature_female` | `af_heart` | A | Adult female, **default narrator** |
| `old_female` | `bf_emma` | B- | Elderly female (British) |
| `young_male` | `am_puck` | C+ | Young male characters |
| `mature_male` | `am_michael` | C+ | Adult male characters |
| `old_male` | `am_fenrir` | C+ | Elderly male, deep voice |
| `child` | `af_sky` | C- | Child characters |

## API Endpoints

### Admin Pipeline (all require `auth:sanctum`)

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/admin/chapters/{id}/pipeline` | Full pipeline visualization |
| POST | `/api/admin/chapters/{id}/pipeline/reprocess` | Re-run from raw |
| DELETE | `/api/admin/chapters/{id}/pipeline/llm` | Clear LLM data |
| GET | `/api/admin/chapters/{id}/tts-preview` | Multi-voice preview |

### Character Management

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/admin/novels/{id}/characters` | List characters |
| POST | `/api/admin/novels/{id}/characters` | Create character |
| PUT | `/api/admin/characters/{id}` | Update character |
| DELETE | `/api/admin/characters/{id}` | Delete character |

### LLM Processing

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/admin/llm/status` | Check OpenAI config |
| POST | `/api/admin/novels/{id}/initialize-characters` | Extract characters |
| POST | `/api/admin/chapters/{id}/llm-process` | Process single chapter |
| POST | `/api/admin/novels/{id}/llm-process` | Batch process |
| GET | `/api/admin/novels/{id}/llm-status` | Processing status |

## Admin UI (chronicle-admin)

### Technology Stack
- **Framework:** Vue 3 + Vite
- **Styling:** Tailwind CSS
- **HTTP:** Axios
- **Routing:** Vue Router
- **State:** Pinia (if needed)

### Pages

1. **Login** (`/admin/login`)
   - Email/password authentication
   - Uses existing Sanctum tokens

2. **Dashboard** (`/admin/`)
   - Novel list with LLM processing status
   - Quick actions

3. **Pipeline Viewer** (`/admin/chapters/{id}/pipeline`)
   - Three-column view: Raw | Processed | LLM
   - Rule applications displayed inline
   - Character speaker highlighting
   - Table detection visualization

4. **Character Manager** (`/admin/novels/{id}/characters`)
   - Character list with voice assignments
   - Edit character details
   - Play voice sample buttons
   - Initialize from LLM button

5. **LLM Processing** (`/admin/novels/{id}/llm`)
   - Chapter processing status grid
   - Batch processing controls
   - Cost estimation

### Deployment
- Served via nginx at `/admin/`
- Static files in Docker volume or built into container
- Separate from Flutter web app

## Infrastructure

### GitHub Secrets
Add to `chronicle-deployment` repository:
- `OPENAI_API_KEY` — OpenAI API key for GPT-4o-mini

### nginx Configuration
```nginx
location /admin {
    alias /var/www/admin;
    try_files $uri $uri/ /admin/index.html;
}
```

### Admin Authentication
- Seeded admin user with role flag
- Reuses Sanctum token authentication
- Admin endpoints check `is_admin` flag

## Implementation Phases

### Phase 1: Foundation (COMPLETE)
- [x] Database migrations (raw_content, characters, llm_paragraphs, summaries)
- [x] Models (NovelCharacter, LlmProcessedParagraph, ChapterSummary)
- [x] Update scraper for raw_content
- [x] Update preprocessor for processed_content
- [x] Admin API endpoints
- [x] LlmProcessingService
- [x] Backfill command

### Phase 2: Admin UI (IN PROGRESS)
- [ ] Vue 3 project setup (chronicle-admin)
- [ ] Admin user seeder
- [ ] Login page
- [ ] Pipeline visualization page
- [ ] Character management page
- [ ] LLM processing controls
- [ ] nginx configuration

### Phase 3: Multi-Voice TTS
- [ ] Update TTS service for multi-voice segments
- [ ] FFmpeg concatenation for voice switching
- [ ] Update paragraph_timings for segments
- [ ] Flutter player updates

### Phase 4: Production Rollout
- [ ] Run migrations on production
- [ ] Backfill raw_content
- [ ] Deploy admin UI
- [ ] Test LLM processing on sample novels
- [ ] Monitor costs

### Phase 5: Flutter Integration
- [ ] Chapter summaries in reader
- [ ] Character profiles viewer
- [ ] Spoiler-aware event timeline
- [ ] Multi-voice playback indicator

## Cost Analysis

### LLM Processing (GPT-4o-mini)
| Operation | Input Tokens | Output Tokens | Cost |
|-----------|--------------|---------------|------|
| Novel init (5 chapters) | ~20,000 | ~1,000 | $0.004 |
| Per chapter | ~4,000 | ~500 | $0.001 |
| 100 chapters | ~420,000 | ~51,000 | $0.09 |
| 1000 chapters | ~4,020,000 | ~501,000 | $0.90 |

### Storage
- Audio: ~6KB/second (~48kbps AAC)
- Chapter average: 5 minutes = ~1.8MB
- 1000 chapters: ~1.8GB S3 storage

## Monitoring

### Metrics to Track
- LLM API costs (OpenAI dashboard)
- Processing time per chapter
- Character detection accuracy
- Dialogue attribution accuracy
- TTS generation queue depth

### Logging
- `LlmProcessingService` logs all API calls with request IDs
- `parse_rule_applications` provides preprocessing audit trail
- `llm_processed_paragraphs.llm_request_id` links to API calls

## Future Enhancements

### Potential Improvements
1. **Voice cloning** — Custom voices for specific characters
2. **Emotion detection** — Adjust TTS parameters for emotional content
3. **Translation** — Multi-language support
4. **Reading speed** — Adaptive speed based on content type
5. **Audiobook chapters** — Music/sound effects at chapter breaks

### Alternative LLM Models
- **Claude 3.5 Haiku** — Better nuance, slightly higher cost
- **Gemini 1.5 Flash** — Cheaper, very long context
- **Local models** — Llama 3 for cost elimination (quality tradeoff)

---

*Last updated: 2026-02-25*
*Status: Phase 2 in progress*
