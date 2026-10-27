---
name: docker-dhi-migration
description: Use this skill when migrating an existing Dockerfile or container image to Docker Hardened Images (DHI), even if the user just asks to harden a container, find a hardened base image, or fix a DHI migration build failure. Covers image mapping, compatibility analysis, Dockerfile changes, and build validation.
license: Apache-2.0
compatibility: Requires Docker with BuildKit for build validation. Access to DHI migration information may require the Docker DHI MCP tools or registry authentication, depending on the image.
---

# Docker Hardened Images Migration

## Overview

Migrate an existing Dockerfile to Docker Hardened Images (DHI) with a deliberate, reviewable plan. Preserve the application's runtime behavior while identifying compatible base images, adapting assumptions that do not hold for hardened images, and validating the result. Use current official migration information rather than guessing image names or tags.

## When to use this skill

Activate this skill when:

- Migrating one or more FROM images in an existing Dockerfile to DHI.
- Finding the DHI equivalent and compatibility requirements for a public or private base image.
- Resolving a build or runtime failure introduced by a DHI migration.
- Reviewing whether a Dockerfile is ready to use a hardened or minimal runtime image.

## Do not use this skill when

Do not use this skill when:

- The user asks a general DHI product or availability question without a Dockerfile migration; use docker-docs to retrieve current official documentation.
- The main task is general image size, cache, or multi-stage optimization unrelated to DHI; use docker-build-strategies.
- The requested artifact is a Docker Agent configuration rather than a Dockerfile.

## Core guidance

### 1. Inspect before editing

1. Find the Dockerfile(s) and read the complete build context relevant to them, including referenced scripts, package manifests, entrypoints, and Compose files when present.
2. Enumerate every FROM instruction, including aliases and platform-specific stages. Also inspect external images referenced by COPY --from=, build arguments used in image references, and Compose image overrides where they affect the deployed image.
3. Record the current base image, tag or digest, stage purpose, package manager, required tools, runtime user, entrypoint, exposed ports, health check, environment variables, and files copied into the final stage.
4. Keep the original Dockerfile intact until a proposed migration has been reviewed. Prefer a focused diff or a separate migrated Dockerfile unless the user explicitly asks to replace the original.

### 2. Look up official migration information

- If the DHI migration tool get_migration_info is available, use it for each applicable source image. Read the returned target image, supported versions, migration notes, and required Dockerfile changes.
- Consult the official [DHI migration documentation](https://docs.docker.com/dhi/migration/) and [DHI Tools documentation](https://docs.docker.com/dhi/tools/) for current workflows and capabilities.
- Never invent a DHI repository, image name, tag, package, or compatibility claim. If an exact mapping cannot be verified, state that it remains unresolved and ask for the missing information or show the official lookup needed.
- Do not assume every builder image has a suitable DHI equivalent. A multi-stage build may keep a conventional builder image and use DHI for the runtime stage when supported and appropriate; explain the decision.
- Preserve version and digest pinning where available. Do not silently switch to an unpinned or moving tag, or choose a different application version merely to make the build pass.

### 3. Plan for compatibility, not just image replacement

Before editing, compare the existing Dockerfile's requirements with the target DHI image and migration notes. Pay particular attention to:

- **Shell and utilities:** minimal or distroless images may not contain a shell, package manager, curl, sh, or debugging tools. Do not add shell-based health checks or RUN commands to a final stage unless the target supports them.
- **Packages and shared libraries:** install build-time packages in the builder stage where possible; verify runtime shared-library and CA certificate requirements rather than assuming they exist.
- **Users and permissions:** verify the target image's configured user and writable paths. Preserve least privilege and explicitly set ownership or permissions when needed.
- **Entrypoint and command:** check exec-form ENTRYPOINT/CMD, signal handling, executable paths, and wrapper scripts. Ensure scripts copied into the image are executable and have compatible interpreters.
- **Artifacts and platforms:** copy only required runtime artifacts from the builder stage; check architecture, libc, native modules, certificates, and other platform-dependent dependencies.
- **Operational behavior:** preserve application ports, environment-variable contracts, startup behavior, and required health checks. Do not claim an image is functionally equivalent from a successful build alone.

### 4. Make a minimal, reviewable change

1. Explain the proposed source-to-target image mapping and any unresolved assumptions before applying non-obvious changes.
2. Update the relevant FROM references and make only the compatibility changes supported by official migration information.
3. Preserve existing build optimizations and stage boundaries where they remain valid. Avoid unrelated refactors, dependency upgrades, or application-code changes.
4. Keep credentials out of Dockerfiles, build arguments, image layers, logs, and the proposed diff. Use the project's approved registry authentication and BuildKit secret mechanisms when needed.
5. If DHI access or private-image authorization is missing, do not attempt to bypass it. Explain which access or credentials the user needs to configure locally.

### 5. Validate the migrated image

Run checks appropriate to the project and available environment:

1. Review the diff and verify every changed image reference against the official migration lookup.
2. Build using the project's documented command, for example docker build --pull -f Dockerfile -t app:dhi-migration ., adjusting the Dockerfile path, build arguments, target stage, and context to match the repository. Do not invent required build arguments or secrets.
3. If the build fails, diagnose the specific incompatibility using the error and official migration notes. Make a minimal fix and rebuild when the environment permits.
4. Run the existing tests and smoke tests. If practical, verify startup, health checks, runtime user, expected ports, and required application behavior.
5. Report what was actually verified. Distinguish a successful image build from successful runtime tests, and call out any checks blocked by unavailable registry access, credentials, platform support, or missing services.

Do not delete images, containers, volumes, or other resources as cleanup without confirming the exact target and consequences with the user.

## Related skills

- docker-build-strategies — multi-stage builds, image size, caching, and general Dockerfile quality.
- docker-docs — current official Docker documentation lookup.

## References

- [Docker Hardened Images migration](https://docs.docker.com/dhi/migration/)
- [Docker Hardened Images tools](https://docs.docker.com/dhi/tools/)
- [DHI migration agent example](https://github.com/docker/docker-agent/blob/main/examples/dhi/dhi.yaml)

## Assets

No bundled assets. Use the repository's Dockerfiles, existing tests, and official DHI migration information as the source of truth.

## Checks

Before returning a migration, confirm that the source Dockerfile was inspected, each proposed target mapping was verified or explicitly marked unresolved, compatibility changes are explained, credentials are not exposed, and build/runtime validation results are reported accurately.
