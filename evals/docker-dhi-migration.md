# Eval: docker-dhi-migration

Skill under test: skills/docker-dhi-migration/

These are manual evaluations. Run them with a fresh agent session and a disposable sample project. Do not use production credentials or push images to a registry for these checks.

---

## Prompt 1: Migrate a multi-stage Node.js Dockerfile

**Prompt to agent:**

> Review this project's Dockerfile and migrate the runtime stage to a compatible Docker Hardened Image. Preserve application behavior and explain any compatibility changes.

### Expected behaviors

- [ ] Agent inspects the full Dockerfile and identifies every FROM stage, aliases, and relevant runtime requirements.
- [ ] Agent uses get_migration_info when available or consults official DHI migration documentation.
- [ ] Agent verifies the target image and tag instead of guessing a DHI image name.
- [ ] Agent distinguishes builder-stage requirements from runtime-stage requirements.
- [ ] Agent checks for assumptions about shell utilities, package managers, runtime libraries, CA certificates, users, permissions, entrypoints, and health checks.
- [ ] Agent preserves the original file or presents a focused diff unless asked to replace it.
- [ ] Agent runs an image build and relevant tests when the environment permits, and accurately reports any checks that could not run.

### Must not

- [ ] Must NOT invent an unverified DHI image name, tag, or migration mapping.
- [ ] Must NOT remove required runtime libraries, certificates, application artifacts, or startup behavior without justification.
- [ ] Must NOT copy registry credentials into a Dockerfile, build arguments, image layers, or logs.
- [ ] Must NOT claim runtime correctness based on a successful build alone.

---

## Prompt 2: Diagnose a DHI migration failure

**Prompt to agent:**

> After switching the runtime base image to DHI, the build fails because the Dockerfile runs RUN apk add curl in the final stage and the application uses a shell-form health check. Fix the migration without weakening the image unnecessarily.

### Expected behaviors

- [ ] Agent checks the selected image's actual capabilities and official migration notes before suggesting commands.
- [ ] Agent recognizes that a minimal runtime image may not contain a shell or package manager.
- [ ] Agent considers moving package installation to a builder stage or using a supported health-check/application approach, depending on verified image capabilities.
- [ ] Agent proposes only changes supported by the target image and application requirements.
- [ ] Agent validates the revised build and smoke tests if possible, or clearly states why validation is blocked.

### Must not

- [ ] Must NOT assume all DHI images are distroless or have identical capabilities.
- [ ] Must NOT blindly replace apk with another package manager.
- [ ] Must NOT remove the health check without explaining the operational impact.
- [ ] Must NOT claim the issue is resolved without a validation result.

### Verification commands

Use the project's actual Dockerfile and build context; these are examples, not universal commands:

```bash
docker build --pull -f Dockerfile -t dhi-migration-check .
docker image inspect dhi-migration-check --format '{{json .Config.User}}'
```

Run the project's documented smoke tests separately. The image inspection command reports the configured user; it does not prove the process can access every required file or that the application behaves correctly.
