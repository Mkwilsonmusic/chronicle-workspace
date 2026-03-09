Authenticate with the Chronicle production API and cache the token.

## Steps

1. **Check for cached token** at `/Users/michaelwilson/.claude/projects/-Users-michaelwilson-chronicle-workspace/memory/api-token`
2. If a cached token exists, **validate it** by running:
   ```bash
   curl -s -o /dev/null -w "%{http_code}" "https://my-chronicle.com/api/user" \
     -H "Authorization: Bearer <TOKEN>" -H "Accept: application/json"
   ```
3. If validation returns `200`, report the token is still valid and stop.
4. If no cached token or validation fails, **authenticate**:
   ```bash
   curl -s -X POST "https://my-chronicle.com/api/login" \
     -H "Content-Type: application/json" -H "Accept: application/json" \
     -d '{"email":"admin@my-chronicle.com","password":"JCCudlPq1MKGoVGVOGsA"}'
   ```
5. Extract the `token` field from the JSON response.
6. **Cache the token** by writing it (just the token string, no newline) to:
   `/Users/michaelwilson/.claude/projects/-Users-michaelwilson-chronicle-workspace/memory/api-token`
7. **Verify** by calling `GET /api/user` with the new token and show the user info.

## Output

Report: token status (reused/new), admin user name, and token expiry info (30-day Sanctum tokens).
