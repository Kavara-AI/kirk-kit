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
   and `cd <version>`.
3. **Key.** Check whether the key is available without reading it:
   - macOS: `security find-generic-password -s kavara-api-key >/dev/null 2>&1 && echo present`
   - otherwise: test whether `KAVARA_API_KEY` is set and non-empty, without printing its value.
   If it is absent, ask the user to create a key in the Kavara console and store it themselves: on macOS with
   `security add-generic-password -a "$USER" -s kavara-api-key -w` (it prompts and does not echo), elsewhere in
   their secrets manager as `KAVARA_API_KEY`.
4. **Install.** `python3 install_kit.py` (use the interpreter from step 1). It verifies every kit file and
   creates `.venv`. Then `source .venv/bin/activate` and `pip install jupyter nbconvert`.
5. **Run.** `jupyter nbconvert --to notebook --execute --allow-errors notebooks/01_rf_quickstart.ipynb --output run.ipynb`
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
| `401 invalid_api_key` | no key reached the API | step 3 |
| `403 request_denied` | the key is unknown or revoked | the user creates a new key at https://kavara.ai/login |
| `429` / `RateLimitedError` | over 300 calls a minute for this account | wait a minute and rerun |
| `422 EnvelopeRefused` outside the refusal step | input not rendered into [-1, 1] | render with `kavara_preprocess.uhura` |
| cannot connect to api.kavara.ai | no network from this environment | run where outbound HTTPS is allowed |

## Going further

- The live limits: `GET /v1/kirk-api/contract` with the key in the `X-Kavara-Key` header.
- The full API specification: `GET /v1/kirk-api/openapi.json` (needs the key).
- The user's own data: set `KIT_IQ_PATH` to a SigMF recording or a `.npy` of complex samples and rerun.
