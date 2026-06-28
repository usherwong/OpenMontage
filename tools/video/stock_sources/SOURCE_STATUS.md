# Stock Source — Live Status Notes

Empirical, network-verified status of the keyless / public stock sources.
Adapters report availability via `is_available()` + optional `runtime_warning`,
surfaced through `source_catalog()` / `source_summary()` (see `__init__.py`).

**Don't trust an adapter's docstring over this file** — docstrings have drifted
from reality before (that's the bug this note exists to prevent). Re-probe with
the snippet at the bottom if you need to confirm, and update this file with the
date when you do.

## Last verified: 2026-06-28

| Source | Result | Reported as | Notes |
|---|---|---|---|
| `coverr` | ❌ keyless; ✅ **200 with valid key + browser UA** | **unavailable** unless `COVERR_API_KEY` | Public API now requires a key (free Demo key, 50 req/hr, self-serve at coverr.co/developers). Gated like `pexels`. **Cloudflare** 403s (error 1010) any request whose User-Agent looks like a default library client (`python-requests`/`python-urllib`), so the adapter sends a browser-like UA — without it even a valid key is rejected. Verified working 2026-06-28 with a real key. |
| `pond5_pd` | ❌ HTTP 403 keyless | **unavailable** unless `POND5_API_KEY` | 403 this run; SSL EOF in earlier testing. Even with a key the public `www.pond5.com` endpoint is unreliable — see `runtime_warning`. |
| `loc` | ⚠️ 403 → **200 with User-Agent** | **available** (+ warning) | Missing UA caused the 403; the adapter now sends an identifying `User-Agent`, which clears it. BUT the `original-format:film/video` facet returned 0 results and the API is slow/rate-limited, so video search is effectively empty. No key to gate on. |
| `archive_org` | ✅ HTTP 200, returns results | available | Working. |
| `wikimedia` | ✅ HTTP 200 | available | Working. |
| `nasa` | ✅ HTTP 200 (~262 KB) | available | Working. |
| `nara` | ✅ HTTP 200 | available | Working. |

Sources gated on their own API keys (`pexels`, `pixabay_video`, `unsplash`,
`mixkit`, `videvo`, `esa`, `noaa`, `dareful`, `jaxa`) report `unavailable` until
their env var is set — that's expected, not breakage.

## Design rule

- A source hard-gated behind credentials → gate `is_available()` on its env var,
  exactly like `pexels`. (Applied to `coverr`, `pond5_pd`.)
- A keyless source that's reachable but flaky/empty → keep it available but set a
  `runtime_warning` class attribute. (Applied to `loc`.) The catalog and summary
  surface it so preflight can warn instead of presenting it as fully healthy.

## Re-probe snippet (stdlib only — no `requests` needed)

```python
import urllib.request, urllib.parse, json
def probe(name, url, params, headers=None, timeout=40):
    req = urllib.request.Request(url + "?" + urllib.parse.urlencode(params), headers=headers or {})
    try:
        with urllib.request.urlopen(req, timeout=timeout) as r:
            print(f"[{name}] HTTP {r.status} OK bytes={len(r.read())}")
    except urllib.error.HTTPError as e:
        print(f"[{name}] HTTPError {e.code} {e.reason}")
    except Exception as e:
        print(f"[{name}] {type(e).__name__}: {str(e)[:90]}")

# coverr (keyless) — expect 403
probe("coverr", "https://api.coverr.co/videos", {"query": "rain city", "page_size": 2})
# loc — 403 without UA, 200 with it
probe("loc no-UA", "https://www.loc.gov/search/", {"q": "film", "fo": "json", "c": 2})
probe("loc +UA", "https://www.loc.gov/search/", {"q": "film", "fo": "json", "c": 2},
      {"User-Agent": "OpenMontage/1.0 (stock source adapter)"})
```
