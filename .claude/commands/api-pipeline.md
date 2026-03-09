Show pipeline stage statuses for all chapters in a novel.

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

## Request

`GET https://my-chronicle.com/api/admin/novels/$ARGUMENTS/pipeline-runs`

Include headers: `Authorization: Bearer <TOKEN>` and `Accept: application/json`

## Output

Display a table of chapters with their 6 pipeline stage statuses:

| Ch# | Title | Scrape | Preprocess | Characters | Dialogue | Narrative | TTS |
|-----|-------|--------|------------|------------|----------|-----------|-----|

Use status indicators: complete, running, failed, pending, skipped.

At the bottom, show summary counts: how many chapters are complete, running, failed, pending for each stage.
