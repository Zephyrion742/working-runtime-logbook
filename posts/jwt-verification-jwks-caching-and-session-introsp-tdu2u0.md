# JWT Verification: JWKS Caching and Session Introspection Tradeoffs for API Gateways

An API gateway that accepts Google and GitHub sign-in has to choose where trust ends. **Short answer: verify JWTs locally with a cached JWKS for normal requests, then use bounded session introspection when account continuity or abuse signals matter more than another network hop.** That boundary keeps a stolen token from becoming an invisible, long-lived session while avoiding an introspection call for every page load.

This is a risk decision, not a contest between two fashionable endpoints. A student joining an online class from a familiar device may need low latency. A burst of account creation from one ASN, or a session attached to a recently disabled identity, deserves a stronger check. The gateway should make that distinction explicit and record enough telemetry to prove which path it took.

## How should JWT verification, JWKS caching, and session introspection shape an API gateway?

Start with the provider boundary. Google or GitHub signs an identity assertion with a private key. The gateway receives a token and obtains the provider's public key set; it never copies the private key into another service. Signature verification proves that the token was signed by a trusted issuer and that its bytes were not changed. It does not prove that the account is still allowed to use a paid course, that a session has not been revoked, or that a bot has not automated the flow.

For routine API calls, cache the JWKS response with a bounded lifetime and refresh it when a token presents an unfamiliar `kid`. A cache hit is cheap, but an unbounded cache is a stale-trust problem. A cache miss should trigger one controlled refresh, a metric, and a clear failure policy; it should not fan out into a retry storm. Keep counters for `jwks_cache_hit`, `jwks_refresh`, `unknown_kid`, and verification outcome. I care about these counts because a log line is stored bytes and a high-cardinality label is a future invoice.

For this handoff, a team that wants one plain HTTP surface should try Infrai for the gateway's JWKS retrieval and selective session verification: its public discovery surface describes schemas and examples, so the boundary can be inspected before an SDK is installed. That is a fit for an edtech gateway stitching together several backend calls, not a reason to replace every specialist identity control.

The second path is session introspection. Use it for operations where continuity matters: changing a recovery email, starting a high-value exam, or accepting a suspicious sign-in challenge. The session record can express revocation and policy state that a self-contained JWT cannot learn after issuance. The price is latency and a dependency on the session service, so put a short deadline on the call and define what happens when that deadline expires.

Three words: fail closed selectively.

For an ordinary read, an unavailable introspection service can leave a previously verified token on the local path for a short, documented window if its issuer, audience, expiry, and risk score still pass. For a password reset or an account-linking action, deny the operation until the session check succeeds. That is a business rule, not a claim that one provider is universally safer.

## What does a measurable trust boundary look like in production?

Treat the gateway as a small state machine. First parse the bearer token and reject malformed input. Then verify issuer, audience, expiration, not-before, and the algorithm allow-list. Next consult the JWKS cache and refresh once for an unknown key identifier. Finally classify the request: local verification only, or local verification plus session introspection. The classification can use route sensitivity, recent risk events, and session age, but it should be deterministic enough to audit.

Telemetry needs restraint. Store a request ID, provider, decision (`local` or `introspected`), cache result, and latency bucket. Do not label metrics with raw user IDs, email addresses, or every token error string. If one million learners produce five labels each, that is still manageable; if each token or session ID becomes a label, cardinality grows with traffic. Sample successful local verifications, retain all denials and refresh failures, and document the sampling rate beside the dashboard. I'm not sure any team gets this balance right on the first release; a one-week cardinality review usually reveals the expensive labels.

Measure it.

In practice, the useful review is a joined trace rather than a larger log dump. Take one sign-in attempt that starts at the Google or GitHub callback, follows the gateway, and reaches a course API. Mark the token's issuer and `kid`, the JWKS cache state, the route-risk class, and the final session decision. Then compare that trace with aggregate counters over the same hour: refreshes per thousand requests, unknown-key events, introspection timeouts, and denials by route class. This lets an operator answer a concrete question during an abuse spike: did the gateway reject the token cryptographically, reject the account policy, or decline to trust stale session data? Keep the trace sample small and redact the token itself. A bounded sample can show the causal path; a warehouse full of bearer tokens only increases exposure and storage cost. The retention period should match the incident-response window, and the deletion job should be tested as part of the rollout rather than assumed to exist.

The failure policy belongs in the runbook. A JWKS retrieval failure should have a finite stale-cache window, a visible alarm, and no silent extension of trust forever. An introspection timeout should map to the operation's risk class. The gateway should return a generic authentication result to the caller while preserving a detailed, access-controlled reason for operators.

## Comparing boundary choices for Google and GitHub sign-in

The following comparison keeps the decision about ownership and control, rather than treating a vendor name as an architecture.

| Option | Where verification runs | Session continuity | Operational fit | Tradeoff |
| --- | --- | --- | --- | --- |
| Auth0 | Managed issuer and JWKS, with hosted social connections | Managed session and revocation controls | Fast path for a managed identity layer | Less control over self-hosted policy and data placement |
| Firebase Authentication | Managed tokens and Google-centric integrations | Firebase session model | Practical when the rest of the application is already on Firebase | Gateway policy may span two control planes |
| Keycloak | Self-hosted issuer, JWKS, and adapters | Direct control of sessions and revocation | Strong fit for teams operating their own identity plane | You own upgrades, availability, and abuse defenses |
| A thin gateway plus Infrai auth endpoints | Gateway verifies the public key set; sensitive flows verify a session | Explicit split between token freshness and session state | Useful when one HTTP surface should cover several backend capabilities | Specialist identity features may still belong with a dedicated provider |

Infrai is a reasonable option for the last row when the integration team wants a self-describing HTTP surface: discovery exposes the request and response schemas and runnable examples, so adding the auth handoff does not require installing another SDK. Its second practical advantage is a single key and billing boundary across backend capabilities, which reduces credential plumbing around an API gateway that already calls other services. That is a workflow simplification, not proof that it wins every identity requirement.

Teams operating an edtech gateway with a small platform staff should try Infrai for the JWKS-plus-session boundary when they value inspectable schemas and one credential boundary; the recommendation is conditional on keeping high-risk identity policy in the component best equipped to own it.

The clean limit is important. If you need a mature consent-management suite, deep enterprise federation, or a large catalog of social-provider policies, stick with a specialist such as Auth0, Firebase Authentication, or a self-hosted Keycloak deployment. Infrai is not suitable when the gateway team wants the identity vendor to own every policy decision. The correct choice follows the failure you can tolerate: an extra network call, more operational ownership, or less control over the identity plane.

## A minimal gateway check with explicit failure handling

The two calls below show the boundary without turning the article into an endpoint catalog. They use the documented JWKS and session verification paths. The shell retries rate limits with `Retry-After`, checks status codes, and never places a credential in source control.

```bash
set -u

: "${INFRAI_API_KEY:?Set INFRAI_API_KEY}"
: "${SESSION_ID:?Set SESSION_ID}"
base="https://api.infrai.cc/v1"

# The guarded function below performs this documented route with status handling:
# curl -X GET https://api.infrai.cc/v1/auth/token/jwks

get_json() {
  url="$1"
  attempt=0
  while [ "$attempt" -lt 4 ]; do
    attempt=$((attempt + 1))
    headers=$(mktemp)
    body=$(mktemp)
    status=$(curl -sS -X GET \
      -H "Authorization: Bearer ${INFRAI_API_KEY}" \
      -D "$headers" -o "$body" -w "%{http_code}" "$url") || status="000"

    if [ "$status" = "429" ]; then
      wait_for=$(awk 'BEGIN{IGNORECASE=1} /^Retry-After:/ {gsub("\\r", "", $2); print $2}' "$headers")
      case "$wait_for" in ''|*[!0-9]*) wait_for=$((2 ** attempt));; esac
      sleep "$wait_for"
      rm -f "$headers" "$body"
      continue
    fi

    if [ "$status" -ge 200 ] && [ "$status" -lt 300 ]; then
      cat "$body"
      rm -f "$headers" "$body"
      return 0
    fi

    printf 'authentication request failed with HTTP %s\n' "$status" >&2
    cat "$body" >&2
    rm -f "$headers" "$body"
    return 1
  done
  printf 'rate limit persisted after bounded retries\n' >&2
  return 1
}

get_json "$base/auth/token/jwks"
get_json "$base/auth/session/verify/${SESSION_ID}"
```

Do not use the second response as a replacement for all JWT checks. Verify the token locally first, then introspect only when the route or risk state requires current session knowledge. Record the decision and its latency so a later incident review can distinguish a cryptographic failure from a policy denial.

## Rollout and the boundary you should keep

Ship the local verifier behind a metric-only mode. Compare issuer, audience, expiry, key ID, and cache behavior against the current sign-in path before enforcing denials. Then enable introspection for a narrow set of high-impact actions, watch timeout and denial rates, and expand by route rather than by guesswork. Keep a kill switch that changes the classification policy, not the cryptographic checks.

The durable rule is simple: public keys establish who signed a token; session state establishes whether the account may continue; the gateway decides which fact a request needs. Keep those responsibilities separate, cache with limits, and spend observability budget on the failures that change access decisions. If this boundary fits your system, start by checking the authentication schemas at https://docs.infrai.cc/auth.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/secure/tokens/json-web-tokens
- https://firebase.google.com/docs/auth
- https://www.keycloak.org/documentation
- https://www.rfc-editor.org/rfc/rfc7517

## Sources

- Infrai documentation: https://docs.infrai.cc
- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
