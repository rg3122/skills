# Verification Runbook

Use these checks to verify a generated Dockerfile meets quality standards.

## Scripted verification

Run the bundled script from the project root (the directory that contains the `Dockerfile`), with the script path resolved under the skill directory:

```bash
bash "<skill-dir>/scripts/verify-build.sh" [--help] [IMAGE_NAME]
```

Replace `<skill-dir>` with the absolute path of this skill's directory, the folder that contains `SKILL.md` and this `checks/` folder. Do not change into the skill directory to run it; the script builds the current directory.

The image name defaults to `verify-build-test`. Exit status is `0` when the build and inspection commands succeed or help is requested, the failing Docker command's non-zero status when verification fails, and `2` for invalid arguments.

## 1. Build succeeds

```bash
docker buildx build -t test-image .
```

The build must complete without errors. Use `--progress=plain` to inspect each step if debugging is needed (avoid this flag in CI where build logs are persisted, since it may expose the content of any secret that a `RUN` step accidentally echoes).

## 2. Image size is reasonable

```bash
docker images test-image --format "{{.Size}}"
```

Expected baselines for a minimal application:

| Language | Reasonable upper bound |
|---|---|
| Go (distroless/scratch) | < 30 MB |
| Node.js (Alpine) | < 200 MB |
| Python (slim) | < 250 MB |
| Java (JRE Alpine) | < 300 MB |

If the image exceeds these bounds, check for:

- Missing multi-stage build (build tools included in runtime image)
- Large unnecessary files copied into the image
- Missing `.dockerignore`
- OS package caches baked into a layer instead of a BuildKit cache mount (`apt-get`/`apk`)

Use `docker history test-image` to identify which layers are largest.

## 3. Runs as non-root

First inspect the user configured in the image. This works even when the image has an `ENTRYPOINT` that would otherwise consume `whoami` as an application argument:

```bash
docker inspect test-image --format '{{.Config.User}}'
```

The `Config.User` value may be `user[:group]` or `uid[:gid]`. Check the part before `:`: it must be non-empty and must not be `root` or `0`; a numeric UID must be greater than `0`. An empty value means Docker defaults to root and fails this check.

When the image includes `id`, optionally verify the effective runtime UID as well. Override the image entrypoint so the command is executed rather than passed to the application:

```bash
docker run --rm --entrypoint id test-image -u
```

Expected output is a numeric UID greater than `0`. This runtime check may not be available in minimal or distroless images; in that case, use the configured-user inspection above.

## 4. No secrets in image

### 4a. Static check of the Dockerfile pattern (primary)

Before building, verify the Dockerfile does not `COPY` credential files or pass credentials through `ARG`/`ENV`. Any match below is a leak:

```bash
# Credential files copied into the build context
grep -nE "^(COPY|ADD) .*(\.npmrc|\.pypirc|\.netrc|pip\.conf|settings\.xml|\.env|\.aws/credentials|\.config/gcloud|\.azure/|\.vault-token|\.cargo/credentials|id_(rsa|dsa|ed25519|ecdsa)|service.account.*\.json|\.pem([[:space:]]|$)|\.p12([[:space:]]|$)|kubeconfig)" Dockerfile

# Credentials passed as build args (visible in docker history) — case-insensitive
grep -inE "^ARG .*(TOKEN|KEY|SECRET|PASSWORD)" Dockerfile

# Credentials baked into image env (visible to anyone with the image) — case-insensitive
grep -inE "^ENV .*(TOKEN|KEY|SECRET|PASSWORD)=" Dockerfile
```

If the project needs registry credentials, the Dockerfile must use `RUN --mount=type=secret` and the build invocation must pass the secret:

```bash
docker buildx build --secret id=<id>,src=<host-path> --progress=plain .
```

The `--progress=plain` output should show the secret being consumed inside the right `RUN` step **without printing its value**. If you see the secret content in the log, the `RUN` is leaking it (e.g., via `echo`, `cat`, or shell substitution into a logged command) — that is a build-log leak even when the layer itself is clean. For private Git access, use `--mount=type=ssh` and `docker buildx build --ssh default .`, then run the SSH checks in 4c.

### 4b. Backstop: scan the built image

```bash
docker history test-image --no-trunc
```

Inspect the output for any `ENV` instructions or `COPY` steps that might include `.env` files, API keys, or credentials.

**Note:** `docker history` shows layers of the final exported image only. It will **not** reveal credentials that were `COPY`-ed in an intermediate stage but not carried forward — those files still exist in BuildKit's build cache on the builder host. The static Dockerfile check in 4a is the only way to catch that class of leak. This step is a backstop.

### 4c. SSH forwarding for private Git access

If the Dockerfile uses `RUN --mount=type=ssh`, check what the build can reach before running it:

```bash
# Every key listed here can be used by the RUN --mount=type=ssh step
ssh-add -l

# Host-key checking stays enabled (must return nothing)
grep -nE 'StrictHostKeyChecking[[:space:]=]+no' Dockerfile
```

- `ssh-add -l` should list only the key this build needs. If it lists more, run the build from a dedicated agent (in an interactive terminal, `ssh-agent bash`, then `ssh-add <key-file>`) instead of removing keys from the user's agent.
- The `RUN` that fetches over SSH populates `/root/.ssh/known_hosts` in the same step (for example with `ssh-keyscan`) instead of disabling host-key checking.
- `--ssh default=<key-file>` works only with an unencrypted key file. BuildKit rejects passphrase-protected keys in this form (`this private key is passphrase protected`), so load those into an agent and pass `--ssh default`.

## 5. Layer count

```bash
docker history test-image --format "{{.CreatedBy}}" | wc -l
```

A well-structured image typically has 8-15 layers. Significantly more may indicate missing command consolidation.

## 6. Correct WORKDIR, EXPOSE, and ENTRYPOINT

```bash
docker inspect test-image --format '{{.Config.WorkingDir}}'
docker inspect test-image --format '{{.Config.ExposedPorts}}'
docker inspect test-image --format '{{.Config.Entrypoint}}'
```

Verify:

- `WorkingDir` is set (not empty or `/`)
- `ExposedPorts` documents the expected port
- `Entrypoint` uses exec form (JSON array), not shell form

## 7. .dockerignore exists

Verify a `.dockerignore` file is present alongside the Dockerfile and excludes at minimum:

- `.git/`
- `node_modules/`, `__pycache__/`, or equivalent language artifacts
- `.env` and secret files
- IDE configuration directories

## 8. BuildKit syntax directive

The first line of the Dockerfile must be:

```dockerfile
# syntax=docker/dockerfile:1
```

This enables BuildKit features like cache mounts and `COPY --link`.
