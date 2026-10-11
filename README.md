# Maven Java version constraints

Minimal reproduction for [Renovate issue #25961](https://github.com/renovatebot/renovate/issues/25961) and [PR #46719](https://github.com/renovatebot/renovate/pull/46719).

## Inputs

The project declares `java.version=1.8` and uses Spring Boot's parent at `2.7.17`.
The configured Maven version range includes `2.7.18` and `3.0.0`.
Their published POMs directly declare `java.version=1.8` and `java.version=17`, respectively:

- [Spring Boot parent 2.7.18](https://repo.maven.apache.org/maven2/org/springframework/boot/spring-boot-starter-parent/2.7.18/spring-boot-starter-parent-2.7.18.pom)
- [Spring Boot parent 3.0.0](https://repo.maven.apache.org/maven2/org/springframework/boot/spring-boot-starter-parent/3.0.0/spring-boot-starter-parent-3.0.0.pom)

## Run both source revisions

Run these commands from a Git checkout of this reproduction repository, with its three files tracked.
The runs were tested with Node.js `24.21.0` and pnpm `11.28.4`.
They use Renovate's public local-platform lookup path and the real Maven Central registry.
No Maven build or GitHub token is required.

First run the unmodified main revision:

```sh
RENOVATE_SOURCE_DIR="$(mktemp -d)"
git clone --single-branch --branch feat/25961-maven-java-constraints-20261003 \
  https://github.com/Boulea7/renovate.git "$RENOVATE_SOURCE_DIR"
git -C "$RENOVATE_SOURCE_DIR" checkout --detach 0fe85b0efc80ff8e0b6bd59d092cf1799e42e0f7
pnpm --dir "$RENOVATE_SOURCE_DIR" install --frozen-lockfile
pnpm --dir "$RENOVATE_SOURCE_DIR" generate
BASE_RUN_DIR="$(mktemp -d)"
LOG_FORMAT=json LOG_LEVEL=debug \
  RENOVATE_BASE_DIR="$BASE_RUN_DIR/base" RENOVATE_CACHE_DIR="$BASE_RUN_DIR/cache" \
  node "$RENOVATE_SOURCE_DIR/lib/renovate.ts" --platform=local --dry-run=lookup
```

Then run the candidate with a separate cache:

```sh
git -C "$RENOVATE_SOURCE_DIR" checkout --detach 6a0f03ae623ce346be0f405819850dd3cca96719
pnpm --dir "$RENOVATE_SOURCE_DIR" install --frozen-lockfile
pnpm --dir "$RENOVATE_SOURCE_DIR" generate
CANDIDATE_RUN_DIR="$(mktemp -d)"
LOG_FORMAT=json LOG_LEVEL=debug \
  RENOVATE_BASE_DIR="$CANDIDATE_RUN_DIR/base" RENOVATE_CACHE_DIR="$CANDIDATE_RUN_DIR/cache" \
  node "$RENOVATE_SOURCE_DIR/lib/renovate.ts" --platform=local --dry-run=lookup
```

Inspect the `packageFiles with updates` JSON log entry for `spring-boot-starter-parent`:

| Source | Extracted Java constraint | Proposed updates |
| --- | --- | --- |
| Main `0fe85b0` | None | `2.7.18` and `3.0.0` |
| Candidate `6a0f03ae` | `1.8` | `2.7.18` only |

Both lookups exit successfully; the proposed updates determine whether the feature works.
Keeping the `2.7.18` upgrade verifies that strict filtering has not simply removed every release.

## Measured lookup cost

The following real-registry runs were measured on 2026-10-11.
Times include CLI startup and lookup, excluding installation and source generation.

| Run | Cache state | HTTP requests | Elapsed time |
| --- | --- | ---: | ---: |
| Main `0fe85b0` | Cold | 5 | 5.764 s |
| Candidate during development, before the final cache-key helper integration and commit | Cold | 295 | 58.626 s |
| Candidate `6a0f03ae` | Same persistent cache populated by the preceding strict run | 1 | 5.317 s |

The cold candidate run enumerated **294 releases** and proposed only `2.7.18`.
The final committed candidate repeated that result with the existing cache.
The final commit was not separately measured with a cold cache.
The datasource enriches releases before `allowedVersions` is applied, so the configured upper bound does not limit the per-release POM requests.
Registry contents, network conditions and cache state can change these measurements.

To repeat a warm lookup, rerun the candidate's `node` command without changing `CANDIDATE_RUN_DIR`.

## Controls and limits

These controls were also run against the candidate during development, each using its own copy of the strict run's populated cache:

- Change `constraintsFiltering` to `none`: both `2.7.18` and `3.0.0` were proposed, with 0 HTTP requests and 4.357 s elapsed.
- Remove the project's `java.version` property while keeping `strict`: both updates were proposed, with 0 HTTP requests and 4.394 s elapsed.
- Change `java.version` to `11` while keeping `strict`: no update was proposed, with 0 HTTP requests and 4.091 s elapsed.

Cold default, `none`, and no-Java controls were not measured.
This feature passes literal Java properties to Renovate's existing strict constraint matching.
It does not implement a general minimum-Java runtime compatibility rule: Java `11` does not match either literal `1.8` or `17` in this example.
Effective remote-parent POM resolution is outside this reproduction.
