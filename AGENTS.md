# AGENTS.md: setting up the Kavara Kirk kit for your user

You are helping a user run Kavara's hosted Kirk API (`https://api.kavara.ai`) with the kit released in this
repository. Follow these steps in order. Stop and report the exact error if any step fails; do not work around
a failed check.

## Rules

1. **The key is the user's secret.** Never print it, log it, echo it, write it to a file, put it on a command
   line, or include it in a message. Read it only from the macOS Keychain entry `kavara-api-key` or from the
   `KAVARA_API_KEY` environment variable of the process that runs the notebook. If neither is set, ask the user
   to store it (step 3) rather than asking them to paste it to you.
2. **Verify before you install.** The tarball hash must match the release notes exactly.
3. **Report what the kit reports.** Give the user the notebook's summary cell verbatim; do not summarise a
   failure as a success.

## Steps

1. **Python.** `python3 --version` must be 3.10 or newer. If it is not, find or install a newer one
   (`python3.12`, `brew install python@3.12`, or `uv python install 3.12`) and use it below.
2. **Download and verify.** From the latest release of this repository, download `kirk-kit-<version>.tar.gz`
   and `SHA256SUMS`. Run `shasum -a 256 kirk-kit-<version>.tar.gz` (or `sha256sum`) and compare it with the
   tarball hash in that release's notes. Stop if it differs. Then unpack: `tar -xzf kirk-kit-<version>.tar.gz`
   and `cd <version>`, and run `shasum -a 256 -c SHA256SUMS` there (every line OK). `SHA256SUMS` lists the
   kit's contents, so it fails if run next to the tarball. GNU tar warnings about `LIBARCHIVE.xattr` headers
   are harmless.
3. **Key.** Check whether the key is available without reading it:
   - macOS: `security find-generic-password -s kavara-api-key >/dev/null 2>&1 && echo present`
   - otherwise: test whether `KAVARA_API_KEY` is set and non-empty, without printing its value.
   If it is absent, ask the user to create a key in the Kavara console and store it themselves: on macOS with
   `security add-generic-password -a "$USER" -s kavara-api-key -w` (it prompts and does not echo), elsewhere in
   their secrets manager as `KAVARA_API_KEY`.
   For the headless notebook run in step 5 the key must be in `KAVARA_API_KEY` in that process's environment.
   If you call the client library directly instead of the notebook, it needs `KIRK_API_KEY` (same value),
   `KIRK_API_URL=https://api.kavara.ai`, `KIRK_API_KEY_HEADER=X-Kavara-Key` and
   `KIRK_API_ROUTE_PREFIX=/v1/kirk-api`; its 0.1.2 defaults do not match the service and give `401`. Load it without it appearing
   in a command line, your output or a file:
   - macOS: `export KAVARA_API_KEY="$(security find-generic-password -s kavara-api-key -w)"`
   - elsewhere: ask the user to run `read -rs KAVARA_API_KEY && export KAVARA_API_KEY` in the shell you will
     use, or to provide it through their secrets manager or CI secret. Never ask them to paste it to you.
4. **Install.** `python3 install_kit.py` (use the interpreter from step 1). It verifies every kit file and
   creates `.venv`. Then `source .venv/bin/activate` and `pip install jupyter nbconvert`.
5. **Run.** Set `KIRK_MODELS_DIR` to a fresh folder so this run does not overwrite earlier saved models, then
   `jupyter nbconvert --to notebook --execute --allow-errors notebooks/01_rf_quickstart.ipynb --output run.ipynb`
6. **Read the result.** Show the user the last cell of `run.ipynb`.

## What success looks like

- Every step in the summary is `passed`: imports, connect, data, learn_score_save_reload, refusal, other_sizes.
- `"all_steps_passed": true` and `"all_other_calls_200": true`.
- Exactly one non-200 call: `POST /v1/kirk-api/learn -> 422 EnvelopeRefused`. That is the deliberate
  refusal of unrendered input, and its message names Uhura.
- `"service_release_id": "kip-customer"`.

## If something fails

| You see | Meaning | Do |
|---|---|---|
| `401 invalid_api_key` | no usable key reached the API: none sent, malformed, or sent as `Authorization: Bearer` | step 3; send it only as `X-Kavara-Key` |
| `403 request_denied` | the relay refuses without saying why: unknown or revoked key, **account not admitted to the Kirk API**, or frozen account | run "Diagnose a 403" below |
| `403 insufficient_scope` | the key is known and admitted but lacks a scope this route needs; the response names it | the user creates a key with that scope |
| `402` | not enough credits for `learn` or `score`; the body is meant to carry `required` and `balance` | report it; the user adds credits; do not retry in a loop |
| `429` / `RateLimitedError` | over 300 calls a minute for this account | wait a minute and rerun |
| `422 EnvelopeRefused` outside the refusal step | input not rendered into [-1, 1] | render with `kavara_preprocess.uhura` |
| cannot connect to api.kavara.ai | no network from this environment | run where outbound HTTPS is allowed |

## Diagnose a 403

Kirk API access needs two things: a valid key and an **admitted account**. Admission is not automatic today;
Kavara admits accounts by hand. `403 request_denied` is deliberately identical for a bad key and an
unadmitted account, so tell them apart with the same key (never print it; always send `X-Kavara-Key`; a
request with no key is blocked at the edge, for Python `urllib` as Cloudflare `403` `error code: 1010`):

```bash
curl -sS -o /dev/null -w '%{http_code}\n' -H "X-Kavara-Key: $KAVARA_API_KEY" https://api.kavara.ai/v1/engine
curl -sS -o /dev/null -w '%{http_code}\n' -H "X-Kavara-Key: $KAVARA_API_KEY" https://api.kavara.ai/v1/kirk-api/contract
```

- `/v1/engine` 200 and `/v1/kirk-api/contract` 403: the key is valid and the account is **not admitted** (or is
  frozen). Stop. Do not retry, rotate the key or work around it. Tell the user to email
  [support@kavara.ai](mailto:support@kavara.ai) from their sign-up address to request Kirk API access, and
  rerun once they say it is granted. Report the notebook's summary as it is: a failure, not a success.
- Both 403: the key is unknown or revoked; the user creates a new key.
- Both 401: no usable key reached the API; step 3.

## Going further

- The live limits: `GET /v1/kirk-api/contract` with the key in the `X-Kavara-Key` header.
- The full API specification: `GET /v1/kirk-api/openapi.json` (needs the key).
- The user's own data: set `KIT_IQ_PATH` to a SigMF recording or a `.npy` of complex samples and rerun.
