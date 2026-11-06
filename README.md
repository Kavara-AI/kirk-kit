# Kavara Kirk kit

Learn from your own tensors on Kavara's hosted Kirk API (`https://api.kavara.ai`), starting with RF.

This repository holds the **released kit** and the instructions for using it. The kit is a Python client, the
renderer that prepares your data (Uhura), and a quickstart notebook. Everything runs on your machine except the
model itself, which runs on Kavara's API under your own key.

> **Using an AI agent?** Point it at this repository and tell it to follow [`AGENTS.md`](AGENTS.md).

## What you need

- Python **3.10 or newer** (`python3 --version`; macOS ships 3.9, so you may need `brew install python@3.12`).
- A Kavara API key: sign in at [kavara.ai](https://kavara.ai/login) and create one. Keep it secret: never paste it
  into a notebook, a script, a command line or a file you share.

## Kirk API access: a key is not enough

A valid key proves who you are. It does not by itself let your account use the Kirk API (`/v1/kirk-api/*`):
the account must also be **admitted**. As of this writing admission is **not automatic**. Kavara admits
accounts by hand, and a new account is not admitted. To request it, email
[support@kavara.ai](mailto:support@kavara.ai) from the address you signed up with and say you want Kirk API
access. Until your account is admitted, every Kirk API call is refused with `403 request_denied`, however
correct your key and code are. `learn` and `score` also spend credits, so an admitted account with no credits
is refused with `402` (see below).

If you are an agent, send `X-Kavara-Key: <key>` on **every** request, including health checks. A request with
no key is refused at the edge: with a default Python `urllib` User-Agent it gets Cloudflare `403` with the
plain-text body `error code: 1010`, otherwise `401 invalid_api_key`. With the key the request passes the
edge. `Authorization: Bearer` is not accepted and gives `401`.

### What the status codes mean

| You see | Meaning | Do |
| --- | --- | --- |
| `401` `{"error":"invalid_api_key"}` | No key reached the API, the key is not in the form Kavara issues, or it was sent as `Authorization: Bearer`. | Send the key in `X-Kavara-Key`. Check it was copied whole. |
| `403` `{"error":"request_denied"}` | The relay refuses and deliberately does not say why. The same body covers an unknown key, a revoked key, an account that is not admitted, and a frozen account. | Run the check below. If it says "not admitted", request admission. Otherwise create a new key at [kavara.ai/login](https://kavara.ai/login). |
| `403` `{"error":"insufficient_scope"}` | The key is known and the account is admitted, but the key lacks a scope this route needs (keys carry `read` and `score`); the response names the scope. | Create a key with the needed scope. |
| `402` | The account does not have enough credits for a `learn` or `score` call. | Add credits. The 402 body is meant to report `required` and `balance`; read it if present. |
| `429` | More than 300 calls a minute for the account. | Wait the `Retry-After` seconds (a minute if it is absent) and retry. |

`403 request_denied` looks the same for a bad key and for an account that is not admitted. Tell them apart by
calling an endpoint that does not need Kirk API admission, with the same key:

```bash
# $KAVARA_API_KEY loaded as in section 3, never typed. The header goes to curl on stdin (-H @-), so the key
# is not in curl's command line; -sS prints errors and the key is not echoed.
printf 'X-Kavara-Key: %s\n' "$KAVARA_API_KEY" | curl -sS -o /dev/null -w '%{http_code}\n' -H @- https://api.kavara.ai/v1/engine
printf 'X-Kavara-Key: %s\n' "$KAVARA_API_KEY" | curl -sS -o /dev/null -w '%{http_code}\n' -H @- https://api.kavara.ai/v1/kirk-api/contract
```

| `/v1/engine` | `/v1/kirk-api/contract` | Meaning |
| --- | --- | --- |
| `200` | `403` | The key is valid. The account is **not admitted** to the Kirk API (or is frozen). Request admission as above. |
| `403` | `403` | The key is unknown or revoked. Create a new key. |
| `401` | `401` | No usable key reached the API. Fix how the key is sent. |
| `200` | `200` | You have access. Rerun; any later `402` is about credits, not admission. |
| any other pair | | Unexpected. Report both codes as printed to [support@kavara.ai](mailto:support@kavara.ai). |

## 1. Download and check

From the [latest release](../../releases/latest), download `kirk-kit-<version>.tar.gz` and `SHA256SUMS`, then:

```bash
shasum -a 256 kirk-kit-0.1.2.tar.gz      # Linux: sha256sum
# must print 8ae2e1f49ce4374f5c0b575f73694bd901989af2790e6aea4d1833bd6b66976e (also in the release notes)
tar -xzf kirk-kit-0.1.2.tar.gz && cd 0.1.2
shasum -a 256 -c SHA256SUMS               # every line must say OK
```

`SHA256SUMS` lists the files **inside** the kit, so check it from the unpacked `0.1.2` folder, not next to the
tarball. `install_kit.py` runs the same check again before installing. GNU tar may print warnings about
unknown extended header keywords (`LIBARCHIVE.xattr…`); they are harmless macOS metadata.

## 2. Install

```bash
python3.12 install_kit.py          # creates .venv in the kit folder; never asks for your key
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install jupyterlab             # optional, for the interactive notebook
```

## 3. Store your key

- **macOS:** store it once in your login Keychain. This prompts for the key and does not echo it:

  ```bash
  security add-generic-password -a "$USER" -s kavara-api-key -w
  ```

- **Windows and Linux, interactive:** enter it at the notebook's hidden prompt; it is kept in memory for that
  session only.
- **Headless notebook runs (scripts, CI, IDEs, agents):** the quickstart notebook reads `KAVARA_API_KEY` from the
  environment first, then the macOS Keychain, then the prompt. Put it there without typing it into a command
  line:

  ```bash
  # macOS, from the Keychain entry above:
  export KAVARA_API_KEY="$(security find-generic-password -s kavara-api-key -w)"
  # Linux or any shell, prompted and not echoed:
  read -rs -p "Kavara API key: " KAVARA_API_KEY && export KAVARA_API_KEY && echo
  # CI: map it from your secrets store into the job's environment as KAVARA_API_KEY.
  ```

  The key is sent as the `X-Kavara-Key` header; an `Authorization: Bearer` header is not accepted.

- **Calling the client library directly (your own code or agent):** in kit 0.1.2 the library does not read
  `KAVARA_API_KEY` and its defaults do not match api.kavara.ai. Set all four, or pass the same values to
  `RemoteBackend(...)` as the notebook does:

  ```bash
  export KIRK_API_KEY="$KAVARA_API_KEY"        # loaded as above, never typed
  export KIRK_API_URL=https://api.kavara.ai
  export KIRK_API_KEY_HEADER=X-Kavara-Key
  export KIRK_API_ROUTE_PREFIX=/v1/kirk-api
  ```

  Without these the library sends the key as `Authorization: Bearer` to the wrong routes and gets `401`. The
  `python -m kirk_remote doctor` tool in 0.1.2 uses those same defaults; do not rely on it yet.

## 4. Run the quickstart

```bash
jupyter lab notebooks/01_rf_quickstart.ipynb
# or headless, always writing the output notebook:
jupyter nbconvert --to notebook --execute --allow-errors notebooks/01_rf_quickstart.ipynb --output run.ipynb
```

It renders sample RF data, creates a model, learns, scores, saves the model, reloads it and scores again (the
answers match), shows the refusal for input that was not rendered, then reruns at other sizes. Every step
reports **passed / failed / skipped**, and a summary always runs at the end. A successful run shows every step
`passed` and one expected non-200: `POST /v1/kirk-api/learn -> 422 EnvelopeRefused` (the deliberate refusal).

## Your data and your models

- **Your own data:** set `KIT_IQ_PATH` to a SigMF recording or a `.npy` of complex samples and rerun. Any
  other tensor works too: put windows on the first axis and render with `kavara_preprocess.uhura.render`.
- **Your models** are saved to `~/kirk-models/`, and a rerun overwrites the previous run's files. Set
  `KIRK_MODELS_DIR` to a new folder per run to keep them apart. They are yours to keep, copy and share;
  they run only through Kavara's API.

## Limits

The live limits are published by the API at `GET /v1/kirk-api/contract` (the notebook reads them). Today:

- **300 calls a minute** per account. Every answer carries `X-RateLimit-Limit`, `X-RateLimit-Remaining` and
  `X-RateLimit-Reset`; over the limit you get `429` with `Retry-After`. The client paces itself and waits
  when asked.
- Matrices up to **256** visible nodes; `complex64` and `complex128` input.
- Every input component (real and imaginary parts separately) must be finite and inside [-1, 1]. Otherwise the
  API answers `422` naming the sample and component, and tells you to render with Uhura.

## The API directly

The kit is a convenience, not a requirement. The API is HTTPS with your key in the `X-Kavara-Key` header. The
full specification is at `GET https://api.kavara.ai/v1/kirk-api/openapi.json` (it requires your key). The input
envelope is the part to leave to the renderer.

## Licence

kirk-client, kirk-remote, the notebooks and the documentation: the Kavara Kirk SDK Licence
([`LICENSE-SDK`](LICENSE-SDK)). kavara-preprocess (Uhura's renderer): Apache-2.0
([`LICENSE-APACHE`](LICENSE-APACHE)). See [`LICENSE`](LICENSE).

## Help

Questions or problems: [support@kavara.ai](mailto:support@kavara.ai).
