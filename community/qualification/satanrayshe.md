# AeglysAI qualification technical evidence: satanrayshe

## Summary

| Item | Value |
|---|---|
| GitHub username | satanrayshe |
| Public fork | https://github.com/satanrayshe/AeglysAI |
| Qualification branch | https://github.com/satanrayshe/AeglysAI/tree/qualification/satanrayshe |
| Upstream commit built and tested | `48047e46aee115181dc0fd67a45105a0b7a45a45` (unmodified; this branch adds only this document and its evidence folder) |
| Build result | **PASS** (`./dev up`, exit 0) |
| Run result | **PASS** (`./dev health` and `./dev demo`, exit 0) |
| Test result | **PASS**: 168 passed, 0 failed, 0 skipped (`./dev test`, exit 0) |
| Evidence | [`satanrayshe-evidence/build-run-test.txt`](satanrayshe-evidence/build-run-test.txt) |

## Environment

- Host: Windows 11, running WSL2 with Ubuntu 24.04.5 LTS (kernel 6.18.54.1-microsoft-standard-WSL2).
- Docker Engine 29.1.3 and Docker Compose 2.40.3 (Ubuntu `docker.io` packages). Buildx 0.30.1 was not installed, so I added the Ubuntu `docker-buildx` package before building.
- git 2.43.0.
- Runtime: .NET 8 inside the repository's own containers, `mcr.microsoft.com/dotnet/sdk:8.0-bookworm-slim` for build and tests and `mcr.microsoft.com/dotnet/aspnet:8.0-bookworm-slim` for the API. `global.json` pins SDK 8.0.204 with `rollForward: latestFeature`. No .NET SDK is installed on the host.

## Setup and .NET 8 prerequisites

The repository's `dev` script needs only Docker with Compose and Buildx. It builds the `api` and `tools` images, runs `AeglysAI.DevTools init /local` to generate a synthetic local configuration and a one-hour local credential under `deployment/docker/.local/` (git-ignored; never copied into this branch), then starts the API and polls `/health`. One image-layer download hit a TLS handshake timeout and was retried automatically; the build still finished with exit code 0.

Commands, in order (full output in the evidence file):

```
./dev up        # build api + tools images, generate local config, start api, wait for health
./dev health
./dev demo
./dev test      # dotnet test AeglysAI.sln -c Release inside the tools container
./dev down
```

Test counts exactly as printed by `dotnet test`:

| Test project | Passed | Failed | Skipped |
|---|---|---|---|
| AeglysAI.ArchitectureTests | 4 | 0 | 0 |
| AeglysAI.Domain.Tests | 14 | 0 | 0 |
| AeglysAI.Application.Tests | 53 | 0 | 0 |
| AeglysAI.Experiments.Tests | 4 | 0 | 0 |
| AeglysAI.IntegrationTests | 93 | 0 | 0 |
| **Total** | **168** | **0** | **0** |

## Architecture understanding

AeglysAI is a .NET 8 Clean Architecture solution in four projects whose dependencies point inward; `tests/AeglysAI.ArchitectureTests` enforces the direction.

- **Domain** holds the authorization vocabulary and invariants: Subject, Resource, Action, Role, Permission, AbacRule and RiskContext.
- **Application** holds the engines and the ports they depend on. RbacAuthorizationEngine is Model A. ModelBAuthorizationEngine with AbacEvaluator is Model B. ModelCAuthorizationEngine with ContextualRiskEngine is Model C. The ports are IRbacPolicyProvider, IAbacContextProvider and IRiskContextProvider.
- **Infrastructure** implements those ports in memory.
- **Api** is an ASP.NET Core minimal API. It validates configuration, freezes policy at startup, authenticates JWT bearer tokens and exposes `/health`, `/identity` and three `/authorization/evaluate` endpoints.

## A/B/C baseline understanding

- **Model A (RBAC):** allows only when one of the subject's roles grants the requested permission; otherwise DENY with a reason.
- **Model B (RBAC + ABAC):** asks Model A first; any denial is final. If A allows, the ABAC conditions are evaluated against a frozen attribute snapshot.
- **Model C (contextual risk):** the subject comes only from the validated token, and the request body may carry only resource and action. Model C asks Model B, which asks Model A; any baseline denial returns DENY. If the baseline allows, the server-owned risk context is scored as a fixed weighted sum of five signals. A missing signal makes the level Unknown and denies. The score maps to ALLOW, STEP_UP, LIMIT or DENY. STEP_UP and LIMIT are recommendations with obligations, not enforcement.
- AuthorizationObservation wraps each evaluation, returns an `X-Correlation-ID` and writes one pseudonymized audit event (subject, resource and action as SHA-256 values).

What `./dev demo` returned on this run:

| Request | Outcome |
|---|---|
| Model A, reports/read | ALLOW, `policyVersion: example-model-a-v1` |
| Model B, reports/read | ALLOW (RBAC PermissionGranted, ABAC ConditionsSatisfied, `snapshotVersion: example-model-b-v1`) |
| Model C, reports/read | STEP_UP, score 0.300, level Medium, obligation VerifyStrongerAuthenticationAndReevaluate |
| Model A, reports/write | DENY, NoMatchingPermission |

The Model C score matches the printed contributions: 0.25×0 + 0.25×0.5 + 0.20×0.5 + 0.15×0.5 + 0.15×0 = 0.300.

## Technical observation

**The audit trail cannot tell which policy version produced a decision.**

API responses carry the RBAC `policyVersion`, the ABAC `snapshotVersion` and, for Model C, the risk `algorithmVersion` and `contextVersion`. The `AuthorizationDecision` audit line written by `LoggingAuthorizationAuditSink` carries EventId, CorrelationId, TraceId, SpanId, the three hashed identifiers, Model, Decision, RiskScore, RiskLevel, Reasons, Timestamp and ProcessingDurationMs, and none of the four versions.

The demo prints a correlation ID for the Model A, B and C requests; all three appear in an `AuthorizationDecision` line, and none of the four lines contains a version label (evidence step 03).

To confirm the gap, I changed the RBAC `Version` in the generated local Development config from `example-model-a-v1` to `example-model-a-v2`, recreated only the `api` container and ran `./dev demo` again (evidence step 04). Every response now reports `policyVersion: example-model-a-v2`, while the new audit lines have exactly the same fields as before. After a policy change, the persistent record of a decision cannot say which policy version made it.

## Why it matters

SPEC-007 asks to "record all reported versions and exact configuration for reproducibility", and SPEC-005 describes a version as "a label, not a content hash or signature". After a configuration change, the only persistent record of a past decision no longer identifies the rules that produced it. That weakens incident review and maps to threat T08 (decision/action mismatch) in the threat model.

## Improvement idea

1. Add the four version labels to `AuthorizationAuditEvent`, the logging sink's message template and the SPEC-008 event contract, with `null` when a stage did not run (for example ABAC and risk for Model A).
2. As a follow-up, log a SHA-256 digest of each frozen policy snapshot once at startup and include it in the event, because equal labels do not prove equal contents.

## Validation approach

- Extend the tests in `AuthorizationAuditTests.cs` so that, for each model, the audit event carries the same versions as the HTTP response for ALLOW, DENY, STEP_UP and LIMIT.
- Assert `null` versions when ABAC or risk is skipped, and the new value after a version change.
- Keep the existing canary tests that prove no token or secret reaches the log.
- Run the full suite (`./dev test`) and repeat the demo-plus-version-change check above.

## Limitations

- Observed on the generated Development configuration with the console logging sink only.
- Leaving versions out of the audit event may be deliberate data minimisation; if so, the startup digest alone would still restore traceability.
- The repository's Python CI and gitleaks jobs were not run.
- No upstream code was changed. Nothing was sent upstream (no pull request, no issue).

## Sanitised evidence

`satanrayshe-evidence/build-run-test.txt` joins steps 00 to 06 (environment, build, health and demo, audit lines, version check, tests, shutdown). ANSI codes and carriage returns were stripped; machine paths and host names were replaced; the generated credential, local config and signed download URLs are excluded. The file contains no tokens, keys, passwords, email addresses or private IP addresses.

## AI assistance disclosure

I used Claude Code (Anthropic) as an assistant. It ran the commands above in my WSL environment, collected and sanitised the logs, and drafted this document. I reviewed every line, checked the observation against the logs myself, and can explain the code paths involved.
