[![CI](https://github.com/dejandvorzak/rest-api-automation/actions/workflows/ci.yml/badge.svg)](https://github.com/dejandvorzak/rest-api-automation/actions/workflows/ci.yml)

# rest-api-automation

REST API test automation for the public [ReqRes](https://reqres.in) demo API, built with **Postman / Newman** and wired into a **Jenkins CI pipeline** running Newman inside a custom Docker image.

This is a portfolio project demonstrating an API testing approach: reusable baseline assertions, real JSON-schema validation, request chaining, positive/negative/edge coverage, and containerised CI reporting.

---

## Setup (API key required)

ReqRes now gates its endpoints behind an API key. The anonymous tier allows only ~40 requests/day per IP, which a 42-request run exhausts immediately (you'll get `429 rate_limit_exceeded`). A **free** account raises this to 250 requests/day and gives you a personal key.

1. Create a free account and generate a key at **<https://app.reqres.in/api-keys>**.
2. In Postman, open the **ReqRes - Demo** environment and set the `apiKey` variable's **current value** to your key. Leave the saved/exported value empty so the key never lands in version control.
3. Make sure the **ReqRes - Demo** environment is selected (top-right) before running.

The key is sent as an `x-api-key` header, configured once at the **collection level** (Authorization tab → API Key → added to Header), so every request inherits it. No request hardcodes the key.

> **Security:** the committed environment file ships with an empty `apiKey` on purpose. Keep your real key local only — never commit it. For CI, inject it as a secret (see the CI section).

---

## What's covered

42 requests across 8 folders, targeting the ReqRes demo endpoints (`/api/users`, `/api/unknown`, `/api/login`, `/api/register`):

| # | Folder | Focus |
|---|--------|-------|
| 1 | Users – List | Pagination, schema validation, `?delay=`, avatar/email format checks |
| 2 | Users – Single | Single reads, 404 handling, malformed ids, chained read |
| 3 | Users – Create | `POST` echo behaviour, `createdAt`, id capture for chaining |
| 4 | Users – Update | `PUT`/`PATCH`, `updatedAt`, chained update |
| 5 | Users – Delete | `204 No Content` contract, empty-body assertion |
| 6 | Resources | Color resource schema, hex validation, pagination, 404 |
| 7 | Auth – Register | Success, missing password (400), token capture |
| 8 | Auth – Login | Success, missing field validation, token type check |

---

## Design decisions

**Baseline assertions at the collection level.** Response-time and JSON content-type checks live in the collection-level `test` script, so every request inherits them without duplication. Request-level scripts only add endpoint-specific assertions.

**Real schema validation.** User and resource objects are validated with `tv4` (bundled in the Postman sandbox) against explicit JSON schemas, rather than probing individual fields with `to.exist`.

**Request chaining via collection variables.** `listUserId`, `createdUserId` and `authToken` are captured at runtime and reused downstream. Chained requests include pre-request fallbacks so each request stays runnable in isolation, while a full top-to-bottom run (Newman's default) exercises the real chain.

**Coverage per resource.** Each resource has positive, negative and edge-case requests — not just happy paths.

**Authentication handled once, at the collection level.** The `x-api-key` header is set on the collection's Authorization tab and inherited by every request, with the key value referenced from an environment variable (`{{apiKey}}`) rather than hardcoded — so the suite stays credential-free in version control.

> **Note on the demo API:** ReqRes write endpoints (`POST`/`PUT`/`PATCH`/`DELETE` on `/api/users`) echo the payload and do **not** persist data. Assertions target status codes, response shape and echoed values accordingly.

---

## Project structure

```
reqres-api-automation/
├── collections/
│   └── reqres-api-tests.postman_collection.json   # the suite (import into Postman)
├── environments/
│   └── reqres.postman_environment.json            # baseUrl + apiKey (empty in VCS)
├── Dockerfile.newman                              # postman/newman + htmlextra reporter
├── Jenkinsfile                                    # CI pipeline (build image + run)
├── package.json                                   # local Newman scripts
├── .gitignore
└── README.md
```

---

## Running locally

### Option A — Postman app

1. Import `collections/reqres-api-tests.postman_collection.json`.
2. Import `environments/reqres.postman_environment.json`, select it, and set `apiKey` to your key (see [Setup](#setup-api-key-required)).
3. Run the collection (Collection Runner), top to bottom.

### Option B — Newman (CLI)

Requires [Node.js](https://nodejs.org).

Newman reads the key from the environment file. Either set `apiKey` locally in
`environments/reqres.postman_environment.json`, or override it at runtime with
`--env-var` so nothing sensitive is written to disk:

```bash
# install Newman + the htmlextra reporter
npm install -g newman newman-reporter-htmlextra

# run, injecting the key at runtime (recommended)
newman run collections/reqres-api-tests.postman_collection.json \
  -e environments/reqres.postman_environment.json \
  --env-var "apiKey=$REQRES_API_KEY"

# or use the npm scripts (expect apiKey to be set in the env file)
npm test          # console output only
npm run test:ci   # JUnit + HTML reports (writes to newman-reports/)
```

### Option C — Newman in Docker

```bash
docker build -f Dockerfile.newman -t reqres-newman:ci .
docker run --rm -v "$(pwd):/etc/newman" -e REQRES_API_KEY reqres-newman:ci \
    run collections/reqres-api-tests.postman_collection.json \
    -e environments/reqres.postman_environment.json \
    --env-var "apiKey=$REQRES_API_KEY" \
    -r cli,htmlextra \
    --reporter-htmlextra-export newman-reports/report.html
```

---

## CI pipeline (Jenkins)

The `Jenkinsfile` defines a pipeline that:

1. **Checks out** the repository.
2. **Builds** a custom Newman image from `Dockerfile.newman` (official `postman/newman:6-alpine` plus the `htmlextra` reporter, which the base image does not include).
3. **Runs** the collection in that container, emitting three reporters: `cli` (console), `junit` (consumed by Jenkins' JUnit plugin for pass/fail trends) and `htmlextra` (archived as a build artifact).

The pipeline runs Newman via explicit `docker build` / `docker run` steps rather than `agent { dockerfile }`, because the Newman image's `ENTRYPOINT ["newman"]` conflicts with the way the Docker Pipeline plugin keeps an agent container alive. Calling `newman run …` directly keeps the entrypoint intact.

The pipeline targets a Docker-in-Docker Jenkins setup (`DOCKER_HOST → tcp://docker:2376`).

### Providing the API key in CI

The key must never be committed. Store it as a Jenkins **Secret text** credential
(e.g. with ID `reqres-api-key`) and expose it to the run, then pass it to Newman
with `--env-var`:

```groovy
environment {
    REQRES_API_KEY = credentials('reqres-api-key')
}
// ... in the run step, add:
//   --env-var "apiKey=$REQRES_API_KEY"
```

This keeps the committed environment file credential-free while the live key is
injected only at run time.

### Reports

- **JUnit XML** → `newman-reports/junit-report.xml` — surfaced natively in the Jenkins build (Test Result trend).
- **HTML (htmlextra)** → `newman-reports/report.html` — archived under build artifacts.

---

## Tech stack

Postman · Newman · newman-reporter-htmlextra · Docker · Jenkins · JavaScript (`pm.*` API) · tv4 (JSON schema)

---

## License

MIT
