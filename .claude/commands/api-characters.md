List characters for a novel with voice assignments and avatar status.

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

1. **Characters**: `GET https://my-chronicle.com/api/admin/novels/$ARGUMENTS/characters`
2. **Avatar status**: `GET https://my-chronicle.com/api/admin/novels/$ARGUMENTS/characters/avatar-status`

Include headers: `Authorization: Bearer <TOKEN>` and `Accept: application/json`

## Output

Display:
- Avatar stats (X/Y characters have avatars)
- Character table with: name, gender, voice (provider + voice name), aliases, avatar status (yes/no), protagonist flag
