# AI Agent Operations Lessons

Date: 2026-06-19

This document collects generic, public-facing lessons for AI coding agents and operators. Each lesson turns an observed failure pattern into an actionable rule for safer migrations, orchestration, validation, and security gates.

## Shared State Keys

**What happened:** One stage wrote a status field to a JSON report, while a later stage read a different key. A failed state was treated as complete, allowing a dangerous next step to proceed.

**The rule:** Define shared JSON state keys once and import them everywhere.

**How to apply:**
- Put cross-stage keys in a constants module or schema file.
- Add tests that write a report and read it through the next stage.
- Treat missing required keys as errors, not defaults.

## Platform-Gated Copy Branches

**What happened:** A Windows-only copy branch ran on Linux because the detection logic found an unrelated alternative tool. The failed subprocess returned a negative exit code that the success check accepted.

**The rule:** Gate OS-specific branches by platform and require subprocess success codes to be non-negative.

**How to apply:**
- Use `sys.platform == "win32"` or `os.name == "nt"` for Windows-only paths.
- Check subprocess success with `0 <= exit_code <= max_success_code`.
- Convert exceptions into explicit failure codes and error records.

## Empty Validation Is Failure

**What happened:** Validation ignored directory-walk errors and accepted a run that checked zero files. An empty or unreadable destination passed as valid.

**The rule:** Validation must record walk exceptions and fail when expected inputs produce zero checked files.

**How to apply:**
- Append `PermissionError` and `OSError` details to validation errors.
- Compare checked file count against the manifest or expected input count.
- Fail validation when a non-empty manifest results in `files_checked == 0`.

## Schema Drift Fails Open

**What happened:** Multiple producers wrote the same JSON intermediate with different field names. A security consumer read only one schema, built zero patterns, and allowed unsafe output while reporting success.

**The rule:** Shared security intermediates must use one schema, and schema mismatches must fail closed.

**How to apply:**
- Keep producer and consumer field names in one schema definition.
- During migrations, read old and new keys explicitly and log which path matched.
- Abort when a required security pattern set is unexpectedly empty.

## Canonical Secret Patterns

**What happened:** Several scanners used separate regex tables for the same secret types. The tables drifted, so one scanner caught a token shape that another missed.

**The rule:** Define each secret regex and severity in one canonical source imported by all scanners.

**How to apply:**
- Create one `secret_patterns` module or data file with name, regex, and severity.
- Let each scanner compile or adapt the shared patterns locally.
- Add tests that assert all scanners load the same pattern names and severities.

## Portable Paths Only

**What happened:** A raw string path containing backslashes was moved to Linux. Backslashes became literal filename characters, and startup failed.

**The rule:** Use forward slashes or path-building APIs for portable source paths.

**How to apply:**
- Prefer `pathlib.Path` or `os.path.join` for constructed paths.
- Search migrated Python source for hardcoded backslashes in paths.
- Treat raw strings as syntax helpers, not cross-platform path normalization.

## Remove Drive-Letter Paths

**What happened:** Stale drive-letter paths remained after a Linux migration. Scripts found no files and quietly indexed nothing.

**The rule:** Remove Windows drive-letter paths during migration and fail fast when required Linux paths are absent.

**How to apply:**
- Search source for `[A-Z]:\\` before declaring migration complete.
- Validate configured roots at startup.
- Raise an error when required roots are missing instead of processing zero files.

## Delegate By Cost

**What happened:** Expensive orchestrator tokens were spent on repetitive review, fanout, and verification work. Cheaper or headless workers could have handled the volume while the orchestrator kept judgment and closure.

**The rule:** Delegate volume work; keep judgment, synthesis, and final closure in the orchestrator.

**How to apply:**
- Classify work as repetitive volume or judgment-heavy before assigning it.
- Send large diffs, search fanouts, and repeated checks to cheaper workers.
- Have the orchestrator spot-check results and make the final decision.

## Adversarial Review After Risky Fixes

**What happened:** A primary fix introduced silent regressions in severity handling and integrity validation. An independent adversarial review found issues that tests missed.

**The rule:** Run an adversarial pass after security-adjacent or irreversible-operation fixes.

**How to apply:**
- Give the reviewer the diff and ask what the fix broke.
- Focus review prompts on regressions, downgraded severity, and weakened gates.
- Add tests for any regression found before closing the work.

## Know Where Filters Apply

**What happened:** A credential guard ignored runtime filter fields because filters were configured at source registration time. The request body looked correct but had no effect.

**The rule:** Verify whether filtering parameters apply at registration time or request time.

**How to apply:**
- Read guard source code when filter behavior affects returned secret fields.
- Test one request that should include a field and one that should omit it.
- Document the correct parameter location next to the integration code.

## Retry With Approval Receipts

**What happened:** An async approval flow issued an approval receipt, but the retry omitted it. The guard treated the retry as a new request and looped back to waiting.

**The rule:** Include approval receipts or request IDs explicitly in retry calls.

**How to apply:**
- Persist the approval receipt returned by the guard or approval channel.
- Pass the receipt in the retry payload under the expected field name.
- Add an integration test for request, approve, retry, and success.

## Config Secrets Are Compromised

**What happened:** A personal access token was found in a plaintext configuration file on disk. It was not committed, but local exposure still made it unsafe.

**The rule:** Revoke any real secret found in plaintext config, even if it was never pushed.

**How to apply:**
- Move secrets to environment files or a secrets manager.
- Revoke and rotate the exposed value immediately.
- Add pre-commit and local scans for YAML, JSON, and config formats.

## Security Gates Fail Closed

**What happened:** A safe-export gate reported success after its redaction builder produced zero patterns. The system believed output was safe while protections were absent.

**The rule:** Security gates must abort on missing, empty, or inconsistent protection inputs.

**How to apply:**
- Treat zero patterns as an error unless explicitly configured for a test fixture.
- Make gate reports include counts for patterns loaded, findings matched, and outputs changed.
- Block export when counts are inconsistent with the input scan.

## Check Existing Service Access

**What happened:** An agent tried to authenticate a CLI tool even though an already-authenticated service connector exposed the needed operations. The extra login was unnecessary.

**The rule:** Check existing connectors before installing or authenticating another access path.

**How to apply:**
- Inventory available MCP servers, connectors, and local credentials first.
- Prefer already-authenticated least-new-access paths for read and management tasks.
- Install or authenticate a CLI only when the existing connector lacks required capability.

## Quick Reference

| Pattern | Rule | Section |
|---|---|---|
| Shared state keys | Define shared JSON state keys once and import them everywhere. | Windows to Linux migration |
| Platform-gated copy branches | Gate OS-specific branches by platform and require subprocess success codes to be non-negative. | Windows to Linux migration |
| Empty validation | Record walk exceptions and fail when expected inputs produce zero checked files. | Windows to Linux migration |
| Schema drift | Use one schema for shared security intermediates and fail closed on mismatches. | Windows to Linux migration; security gates |
| Secret pattern drift | Import every scanner pattern from one canonical source. | Windows to Linux migration; security gates |
| Portable paths | Use forward slashes or path-building APIs for portable source paths. | Windows to Linux migration |
| Drive-letter leftovers | Remove drive-letter paths and fail fast when required Linux paths are absent. | Windows to Linux migration |
| Delegation cost | Delegate volume work and keep judgment, synthesis, and closure in the orchestrator. | AI orchestration |
| Adversarial review | Run an adversarial pass after security-adjacent or irreversible-operation fixes. | AI orchestration |
| Filter location | Verify whether filtering parameters apply at registration time or request time. | Secret management |
| Approval retry | Include approval receipts or request IDs explicitly in retry calls. | Secret management |
| Plaintext config secret | Revoke any real secret found in plaintext config, even if never pushed. | Secret management |
| Fail-closed gates | Abort on missing, empty, or inconsistent protection inputs. | Security gates |
| Existing service access | Check existing connectors before installing or authenticating another access path. | Service access |
