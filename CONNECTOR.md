# Product Hunt

- `custom.producthunt` connects Muse to Product Hunt through its official hosted MCP server at `https://mcp.producthunt.com/mcp` (server name producthunt-mcp, version 1.0.1). It authenticates with the registered `oauth2_code` credential and calls only `mcp.producthunt.com` and `www.producthunt.com`.

- The one helper is `~/workspace/custom-connectors/custom.producthunt/bin/client.py`. It speaks Streamable HTTP MCP, carries the credential as a bearer token, and exposes `list-tools` and `call-tool` commands.

## Connect

- The connection was registered with `credentials.request_api_access` and these details. Reuse them to reconnect:

```json
{
  "provider": "producthunt",
  "auth_scheme": "oauth2_code",
  "issuer_domain": "producthunt.com",
  "authorization_url": "https://www.producthunt.com/mcp/oauth/authorize",
  "token_url": "https://www.producthunt.com/mcp/oauth/token",
  "token_endpoint_auth_method": "none",
  "scopes": "mcp:account launches:read launches:write ads:read ads:write",
  "resource": "https://mcp.producthunt.com/mcp",
  "revoke_url": "https://www.producthunt.com/mcp/oauth/revoke",
  "registration_url": "https://www.producthunt.com/mcp/oauth/register",
  "api_hosts": ["mcp.producthunt.com", "www.producthunt.com"]
}
```

- Before the first change in a conversation, prove access with a harmless read:

```text
python3 ~/workspace/custom-connectors/custom.producthunt/bin/client.py call-tool --name get_my_account --arguments-file <path-to-json-with-empty-object>
```

- The protected-resource metadata lives at `https://mcp.producthunt.com/.well-known/oauth-protected-resource` and the authorization-server metadata at `https://www.producthunt.com/.well-known/oauth-authorization-server/mcp/oauth`. Read them again if a reconnect needs fresh endpoints.

## Credentials

- The registered connection holds the credential. Helpers import `/opt/muse/skills/custom-connector-setup/bin/dynamic_credentials.py`, which returns an `hsurr:` surrogate and places it where the provider reads the key. Muse swaps in the real credential only on an approved request to a registered host.

- Never put a key, token, password or cookie in this doc, a helper, a command argument, an environment variable or a log. Never ask the user to paste one in chat.

- Keep redirects disabled on authenticated requests.

- On a 401 or 403, check the request first. Confirm that the helper attached the credential, that the host is registered and that the placement and scopes match.

- Only when a request that carried the credential is still rejected, call `credentials.request_api_access` with the details above and `"reconnect": true`.

## Call a tool

- Write each call's arguments as one JSON object in a workspace file. Keep credentials out of that file.

```text
python3 ~/workspace/custom-connectors/custom.producthunt/bin/client.py list-tools
python3 ~/workspace/custom-connectors/custom.producthunt/bin/client.py call-tool --name <tool> --arguments-file <path-to-json>
```

- `list-tools` returns every tool with its input schema in one page (verified 2026-11-09, no cursor). Run it again after a server update, because only names listed in `ALLOWED_TOOLS` in client.py can be called, and a tool the server adds later stays blocked until reviewed and added.

- A transport success is not a tool success. The client exits with status 1 when the result carries `isError`, and it reports a JSON-RPC error without replaying the call.

## Verified account read

- `get_my_account` reads the authenticated account. It takes an empty arguments object and returns the account id, username, display name and granted scopes.

- Verified 2026-11-09 against the user's account (username wbic16) with all five scopes granted.

## Tool catalog

- Account and launches (read): `get_launch_context`, `list_my_launches`, `get_launch`. `get_launch_context` reads composer limits, account eligibility, launch-date availability, and topic, category, maker, backer and shoutout suggestions. These are untested beyond their schemas.

- Launch changes: `start_launch`, `update_launch`, `submit_launch`, `prepare_launch_asset`. These create or change the user's own launches and need the user's authorization for each change. All untested.

- Discovery and research (read, no account scope needed): `get_categories`, `get_products`, `find_alternatives`, `get_product_profile`, `search_launches`, `semantic_product_search`, `get_leaderboard`. All untested.

- Investor sourcing (read): `find_investable_startups`. Untested.

- Feedback: `submit_feedback` sends product feedback to Product Hunt about the MCP experience. The server instructions ask for it once per task when friction appears. Send it only when there is real friction to report, and never include secrets or personal details.

## Recover from errors

- The client follows no redirects and retries nothing. A connection that stops responding for 45 seconds fails without a retry.

- After a failed or timed-out change, check the current state with a read before trying again, because the first request may have completed.

- The endpoint advertises a rate limit of 250 requests per 900 second window in its response headers (observed 2026-11-09). Keep call volume close to ordinary use.

## Limits

- The connector reaches only what the MCP server exposes. It cannot browse Product Hunt pages, vote, comment or follow users, because the server offers no such tools.

- Write tools are allowlisted in the client but every launch change still needs the user's authorization in the conversation.

## Related skills

- Use `muse.skill_search` to find other skills and `muse.connector_search` to find other connectors.
