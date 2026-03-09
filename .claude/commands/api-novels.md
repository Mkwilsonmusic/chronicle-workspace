List or search novels from the Chronicle production API.

**Argument**: $ARGUMENTS (optional search query)

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

- If `$ARGUMENTS` is empty: `GET https://my-chronicle.com/api/novels/subscribed`
- If `$ARGUMENTS` is provided: `GET https://my-chronicle.com/api/novels/search?q=$ARGUMENTS`

Include headers: `Authorization: Bearer <TOKEN>` and `Accept: application/json`

## Output

Display a table of novels with: ID, title, author, chapters count, and subscription status.
