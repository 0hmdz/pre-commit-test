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
