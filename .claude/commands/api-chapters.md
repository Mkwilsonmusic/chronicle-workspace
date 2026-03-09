Get chapter detail including paragraphs, TTS status, and pipeline info.

**Argument**: $ARGUMENTS (chapter ID — required)

## Auth Preamble

1. Read cached token from `/Users/michaelwilson/.claude/projects/-Users-michaelwilson-chronicle-workspace/memory/api-token`
2. Validate with `GET https://my-chronicle.com/api/user` (check for HTTP 200)
3. If invalid or missing, authenticate:
   ```bash
   curl -s -X POST "https://my-chronicle.com/api/login" \
     -H "Content-Type: application/json" -H "Accept: application/json" \
     -d '{"email":"admin@my-chronicle.com","password":"JCCudlPq1MKGoVGVOGsA"}'
   ```
   Extract token and write to the cache file.

## Requests

Make these API calls (can be parallel):

1. **Paragraphs**: `GET https://my-chronicle.com/api/chapters/$ARGUMENTS/paragraphs`
2. **TTS status**: `GET https://my-chronicle.com/api/chapters/$ARGUMENTS/tts`
3. **Pipeline**: `GET https://my-chronicle.com/api/admin/chapters/$ARGUMENTS/pipeline/summary`

Include headers: `Authorization: Bearer <TOKEN>` and `Accept: application/json`

## Output

Display:
- Chapter title and paragraph count
- TTS status (pending/generating/complete/failed), voice, duration
- Pipeline stage statuses (scrape, preprocess, characters, dialogue, narrative, tts)
- First 3 paragraphs as a content preview (truncated to ~200 chars each)
