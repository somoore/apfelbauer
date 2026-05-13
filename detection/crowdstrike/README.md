# CrowdStrike Falcon Detection Rules

Detection rules for CrowdStrike Falcon Event Search covering the Apple Intelligence enablement primitives and related framework-access patterns described in the project [README](../../README.md).

**Status: untested in a production Falcon environment.** I do not have a Falcon console or sensor with live macOS endpoint telemetry to validate these against. The FQL queries follow Event Search syntax and use common CrowdStrike field names (`event_simpleName`, `ImageFileName`, `CommandLine`, `ParentBaseFileName`), but the canonical field for "Apple-signed code" varies by sensor version — see open questions below. Review, tune, and test against a sample of `event_simpleName=ProcessRollup2` data before enabling, and please share back any field-name corrections. **Use at your own risk.** Provided as-is, with no warranty of any kind. See [LICENSE](../../LICENSE).

## Open Questions

- **Apple-signer field for Rule 4.** Rule 4 (FoundationModels load from unsigned process) uses `SignInfoFlags!=*Apple*` to filter out Apple-signed loaders. **Verify against your sensor schema.** The canonical CrowdStrike field for "Apple-signed" varies across sensor versions: `SignInfoFlags`, `AuthenticodeHashData`, `CertificatePublisher`, or per-event signer fields. Inspect a sample `event_simpleName=ProcessRollup2` for a known-Apple-signed binary in your tenant and confirm which field is populated. Replace if needed and share back so I can correct the rule.
- **ImageLoad telemetry availability.** Rule 4 requires the Falcon sensor to be configured to capture library/dylib loads. Verify under sensor policy that `ImageLoad` telemetry is enabled — many production policies have this off by default for performance.
- **OverWatch overlap.** CrowdStrike OverWatch may already independently flag some of these patterns (rapid shell exec from Swift, defaults writes to security-relevant domains). If you can confirm what OverWatch already catches on its own, I can scope the custom IOAs to fill only the gaps.

## Deployment

### Via Falcon Console Custom IOAs

1. Navigate to **Configuration** > **Custom IOA Rule Groups**
2. Click **Create rule group**, name it `apfelbauer - Apple Intelligence Abuse`
3. Set platform to **Mac**
4. For each rule in `apfelbauer-rules.json`:
   - Click **Add new rule**
   - Select the appropriate **Rule Type** (Process Creation, etc.)
   - Set **Action**: **Detect** (recommended for initial deployment)
   - Set **Severity** per the JSON
   - Configure the rule fields to match the FQL query logic
5. Enable the rule group and assign it to the appropriate **Prevention Policies**

### Via Event Search (Threat Hunting)

1. Navigate to **Investigate** > **Event Search**
2. Paste the `query` field from each rule object
3. Set the time range (recommended: last 7 days for initial sweep)
4. Save as a **Saved Search** for recurring use

### Via Falcon Fusion SOAR

For automated response workflows:
1. Create a new workflow in **Fusion** > **Workflows**
2. Set the trigger to **Detection** with the custom IOA rule name
3. Add response actions (e.g., contain host, notify SOC, create ticket)

## Query Syntax

Rules use Falcon Query Language (FQL) for Event Search. Key fields:

- `event_simpleName` filters event type (ProcessRollup2, DnsRequest, etc.)
- `ImageFileName` matches the process binary path
- `CommandLine` matches the full command line
- `ParentImageFileName` / `ParentBaseFileName` match the parent process
- Wildcards (`*`) are supported in string fields
- `|` (pipe) separates filter from aggregation

## Tuning

- **Rule 5 (Swift non-Xcode parent):** Add exclusions for known CI/CD systems: append `ParentBaseFileName!=["fastlane","mint","tuist"]` to the query
- **Rule 4 (FoundationModels load):**
  - ImageLoad events require the Falcon sensor to capture library loads. Verify with your Falcon admin that `ImageLoad` telemetry is enabled in the sensor policy.
  - The query uses `SignInfoFlags!=*Apple*` as a code-signer filter. **Verify against your sensor schema** — the canonical CrowdStrike field for "Apple-signed" code varies by sensor version (`SignInfoFlags`, `AuthenticodeHashData`, `CertificatePublisher`, or per-event signer fields). If `SignInfoFlags` is not populated in your environment, replace with the field your sensor actually exposes (inspect `event_simpleName=ProcessRollup2` sample events in Event Search to confirm).
- **Rule 7 (Rapid shell exec):** Adjust the `count` threshold and `dccount` minimum based on your environment's baseline Swift usage.

## Notes

- FQL queries work in both Event Search and the Streaming API.
- Falcon Data Replicator (FDR) customers can run these queries against their SIEM/data lake for longer retention.
- CrowdStrike OverWatch may independently flag some of these patterns; these rules ensure coverage for automated detection.
