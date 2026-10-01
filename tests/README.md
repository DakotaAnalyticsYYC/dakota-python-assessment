# tests/: pytest Tests

**Expected coverage at minimum:**
- Token refresh when a token expires mid-run
- Retry/backoff on 429 (honouring `Retry-After`) and 5xx; no retry on other 4xx
- Pagination reads every page
- Cleaning and type coercion of messy records
- Idempotent load (running twice gives the same state)
- At least one API test (e.g. 401 without a token, `updated_since` filtering)
- Writeback validation rules
- Silver history: a restated record creates a new version; reloading unchanged data does not

Run with `run.bat test` (inside a container).
