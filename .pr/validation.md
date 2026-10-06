# SDK #4976 validation

Refreshed and validated locally on 2026-10-05 against upstream `cb7eabaf7`:
macOS arm64, Python 3.13.3, LiteLLM 1.93.0, SDK 1.53.0.
No dependency or lockfile changes relative to upstream.

## Regression coverage

Permanent tests under `tests/sdk/llm/` cover completion selection, generic
LiteLLM output-item events, sync/async streams, output reconstruction without
callbacks, tokenizer shapes/fallbacks, and message metadata compatibility.
The six generic-event reconstruction cases failed before the adapter fix and
passed afterward, preserving the original output-item identity.
Conflict resolution retains upstream async hard/idle timeouts and transient
error classification. A new regression verifies ignored provider events reset
the idle timer without emitting callbacks.

## Results

| Check | Result |
| --- | --- |
| Full SDK suite plus local development matrix | 8,175 passed, 1 failed, 9 skipped, 12 xfailed |
| ACP session-persistence group rerun | 39 passed; isolated failing test also passed |
| Local development matrix alone | 1,285 passed, including 240 generic-event reconstruction cases |
| Real local HTTP/SSE transport checks | 12/12 passed |
| Token counting with/without Transformers | 2/2 passed |
| Rebuilt packaged Agent Server | `/health` returned HTTP 200; owned process stopped |
| Repository-wide pre-commit and staged commit hooks | Passed |
| Dynamic-access baseline and diff whitespace checks | Passed |
| Package-version guard against refreshed upstream | Passed; no version changes detected |

The combined run completed in 323.84s with 122 warnings. Its sole failure was
`TestACPSessionIdPersistence.test_mask_callback_does_not_retain_agent`, which
timed out waiting for garbage collection. It passed in isolation (0.16s), and
all 39 tests in its surrounding group passed on rerun. The test and ACP
implementation are unchanged from upstream. This suggests a cleanup/timing
flake, but the broad run is not recorded as all-green. No LLM or matrix case
failed. Warnings include deprecations, provider-cost and async cleanup warnings.

Live provider calls were skipped because no provider credentials were configured.
Local transport checks exercise real LiteLLM HTTP/SSE parsing, not a live model.
Development-only matrix/smoke scripts and detailed reports are retained locally,
not included in this branch's final diff.

## Reproducible repository checks

```sh
make build
uv run pytest tests/sdk/llm/test_llm_completion.py \
  tests/sdk/llm/test_responses_parsing_and_kwargs.py -q -o addopts='--tb=short'
uv run pytest tests/sdk -q -o addopts='--tb=short'
uv run pre-commit run --all-files
uv run python scripts/check_forbidden_dynamic_attributes.py --baseline-ref upstream/main
git diff --check
make build-server
VERSION_BUMP_BASE_REF=upstream/main uv run python .github/scripts/check_version_bumps.py
```

The local matrix and smoke-check counts above are recorded evidence; their
development harnesses are not required to run the permanent regression tests.
