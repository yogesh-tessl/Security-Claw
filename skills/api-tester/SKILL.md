---
name: api-tester
description: "Performs automated security assessments of REST and GraphQL API endpoints including authentication bypass, authorization flaws (BOLA/BFLA), injection attacks, schema introspection abuse, batch query attacks, and business logic flaws. Supports OpenAPI/Swagger spec import for endpoint discovery. Maps findings to OWASP API Security Top 10 (2023) and OWASP GraphQL Cheat Sheet. Use when testing API endpoints for security vulnerabilities, assessing authorization controls, or performing OWASP-aligned API penetration testing."
metadata:
  {
    "openclaw":
      {
        "emoji": "🔌",
        "requires": { "bins": ["python3", "curl"] },
        "install":
          [
            {
              "id": "pip-api-tester-deps",
              "kind": "shell",
              "cmd": "pip3 install requests httpx gql[requests] pyyaml jsonschema rich",
              "bins": [],
              "label": "Install API tester dependencies (pip)",
            },
          ],
      },
  }
---

# REST & GraphQL API Security Testing Skill

## Workflow

1. **Scope authorization** — Confirm the target is in scope and the agent has written authorization before any active testing.
2. **Import spec** — Parse OpenAPI/Swagger spec to enumerate endpoints, methods, parameters, and auth schemes.
3. **Test authentication** — Probe auth mechanisms (JWT, OAuth, API keys) for weaknesses.
4. **Test authorization** — BOLA/BFLA testing across all discovered endpoints.
5. **Test injection** — SQLi, NoSQLi, command injection, SSTI across REST and GraphQL surfaces.
6. **Test resource consumption** — Rate limiting, batch abuse, deep query DoS.
7. **Validate findings** — Confirm true positives by reproducing each finding. Discard false positives before reporting.
8. **Report** — Map confirmed findings to OWASP API Security Top 10 (2023) references.

**Safety checkpoint:** Before any destructive or state-changing test (PUT/DELETE, mutation, brute-force), confirm the operation is authorized and will not damage production data.

---

## REST API Testing

### Endpoint Discovery from OpenAPI/Swagger Spec

Parse OpenAPI 3.x or Swagger 2.x specs to extract all routes, methods, parameters, and authentication schemes. Flag deprecated and unauthenticated endpoints.

**OWASP:** API9:2023 Improper Inventory Management

```bash
curl -s https://api.target.com/swagger.json | python3 -c "
import json,sys
spec=json.load(sys.stdin)
for path,methods in spec.get('paths',{}).items():
    for method in methods:
        print(f'{method.upper():8} {path}')
"
```

---

### Authentication Testing

Probe authentication mechanisms for weaknesses.

**Tests:**
- Missing auth on protected endpoints
- JWT: `alg:none`, HS256/RS256 confusion, expired token acceptance
- OAuth: token leakage, open redirect, PKCE bypass
- API key in URL parameters (logged/cached)
- Brute-force with no rate limit

**OWASP:** API2:2023 Broken Authentication

---

### Authorization Testing (BOLA / BFLA)

Test every endpoint for object-level and function-level authorization failures.

**Tests:**
- **BOLA (API1:2023):** Swap resource IDs between accounts
- **BFLA (API5:2023):** Low-privilege token on admin endpoints
- **Horizontal:** User A accesses User B's objects (same role)
- **Vertical:** Regular user accesses admin-only endpoints

```bash
# BOLA test — swap IDs with different auth tokens
curl -H "Authorization: Bearer TOKEN_B" https://api.target.com/api/v1/users/USER_A_ID
```

**Validation:** Confirm the response contains User A's data, not a 403/404.

---

### Excessive Data Exposure

Identify responses returning more fields than necessary.

**Tests:**
- Compare returned fields vs client-rendered fields
- Check for PII, internal IDs, password hashes in responses
- Test `fields`/`include` filter bypass
- Probe for active debug endpoints

**OWASP:** API3:2023 Broken Object Property Level Authorization

---

### Mass Assignment Testing

Test whether endpoints accept and apply unexpected fields.

```bash
curl -X PUT https://api.target.com/api/v1/profile \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "test", "role": "admin", "isAdmin": true, "credits": 99999, "verified": true}'
```

**Validation:** Check if privileged fields (`role`, `isAdmin`, `credits`) were persisted by re-fetching the resource.

**OWASP:** API6:2023 Unrestricted Access to Sensitive Business Flows

---

### Rate Limiting & Resource Consumption

Test for missing rate limits and resource abuse vectors.

**Tests:**
- Concurrent request flooding
- Large payload injection (body size, array sizes)
- Deep pagination abuse (`?page=999999`)
- Regex DoS via crafted inputs

```bash
for i in {1..100}; do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST https://api.target.com/login \
    -d '{"user":"admin","pass":"test"}' &
done
wait
```

**OWASP:** API4:2023 Unrestricted Resource Consumption

---

### Injection Testing (SQLi, NoSQLi, Command Injection)

Test REST parameters for injection across query params, JSON body, and path params.

**Payloads:**
- SQL: `' OR 1=1--`, `" UNION SELECT username,password FROM users--`
- NoSQL: `{"$gt": ""}`, `{"$ne": null}`
- Command: `; ls -la`, `| cat /etc/passwd`
- SSTI: `{{7*7}}`, `${7*7}`

**Tools:** `sqlmap`, custom `httpx` probes

---

### Security Headers & Transport Layer

Validate API security posture at transport and HTTP layer.

**Checks:**
- HTTPS enforced (no HTTP fallback)
- `Strict-Transport-Security` header present
- `Content-Type: application/json` enforced
- CORS policy (`Access-Control-Allow-Origin: *` on credentialed endpoints)

**OWASP:** API8:2023 Security Misconfiguration

---

## GraphQL API Testing

### Schema Introspection

Enumerate the full GraphQL schema to discover types, queries, mutations, and subscriptions.

```bash
curl -s -X POST https://api.target.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name fields { name } } } }"}' | python3 -m json.tool
```

**OWASP:** API8:2023 Security Misconfiguration (introspection enabled in production)

---

### Introspection Bypass

Enumerate schema when `__schema` introspection is disabled.

**Techniques:**
- `__type` query (often not blocked separately)
- Field suggestion exploitation (typo triggers server correction with valid field names)
- `__typename` meta-field on all types
- Clairvoyance tool for wordlist-based field discovery

```bash
# Misspell a field to trigger "Did you mean: secretField?"
curl -s -X POST https://api.target.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ user { passsword } }"}'
```

---

### GraphQL Injection

Test GraphQL arguments for SQL, NoSQL, and OS command injection.

```graphql
# SQL injection via arguments
{ user(email: "admin'--") { id name email } }
{ user(id: "1 UNION SELECT username,password FROM users--") { id } }

# NoSQL injection
{ user(filter: "{\"$gt\": \"\"}") { id } }
```

**OWASP:** API3:2023, A03:2021 Injection

---

### Batch Query Attack (DoS / Brute-force)

Abuse GraphQL query batching to bypass rate limits or amplify requests.

```json
[
  { "query": "mutation { login(email: \"admin@test.com\", password: \"pass1\") { token } }" },
  { "query": "mutation { login(email: \"admin@test.com\", password: \"pass2\") { token } }" },
  { "query": "mutation { login(email: \"admin@test.com\", password: \"pass3\") { token } }" }
]
```

**Validation:** Check if all mutations execute in a single HTTP request (rate limit bypass). Confirm whether the server enforces per-query or per-batch limits.

**OWASP:** API4:2023 Unrestricted Resource Consumption

---

### Deep Query / Circular Query DoS

Send deeply nested or circular queries to test resource exhaustion controls.

```graphql
{ user { friends { friends { friends { friends { id name friends { friends { id } } } } } } } }
```

**Checks:** Query depth limit, complexity scoring, timeout controls, cost analysis.

**OWASP:** API4:2023 Unrestricted Resource Consumption

---

### GraphQL Authorization (BOLA / Vertical Escalation)

Test whether resolvers enforce authorization per field and operation.

**Tests:**
- Access another user's private data by changing ID arguments
- Invoke admin mutations with a regular user token
- Access hidden fields through direct field argument injection

**Validation:** Confirm unauthorized data is returned (not just a 200 with empty/null fields).

**OWASP:** API1:2023 BOLA, API5:2023 BFLA

---

### Subscription Security

Test GraphQL subscriptions for unauthorized data streaming.

**Tests:**
- Subscribe to another user's events without authorization
- Test subscription filtering bypass
- WebSocket authentication (token at connection vs per-message)
