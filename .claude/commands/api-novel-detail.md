Get full detail for a novel including chapters and pipeline status.

**Argument**: $ARGUMENTS (novel ID — required)

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

1. **Novel detail**: `GET https://my-chronicle.com/api/novels/$ARGUMENTS`
2. **Chapters**: `GET https://my-chronicle.com/api/novels/$ARGUMENTS/chapters`
3. **Pipeline runs**: `GET https://my-chronicle.com/api/admin/novels/$ARGUMENTS/pipeline-runs`

Include headers: `Authorization: Bearer <TOKEN>` and `Accept: application/json`

## Output

Display:
- Novel metadata (title, author, status, rating, genres, chapter count)
- First/last 5 chapters with TTS status
- Pipeline summary (how many chapters at each stage)
