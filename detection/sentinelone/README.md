# SentinelOne Detection Rules

Detection rules for SentinelOne Deep Visibility / STAR (Storyline Active Response) covering the Apple Intelligence enablement primitives and related framework-access patterns described in the project [README](../../README.md).

**Status: untested in a production SentinelOne environment.** I do not have a Singularity console or agent with live macOS endpoint telemetry to validate these against. The queries follow Deep Visibility / PowerQuery syntax and use common field names (`EventType`, `SrcProcName`, `SrcProcCmdLine`, `SrcProcParentName`, `TgtProcName`), but sensor field availability varies by agent version and policy. Review, tune, and test against a small site or account scope before enabling broadly, and please share back any false-positive patterns or query corrections. **Use at your own risk.** Provided as-is, with no warranty of any kind. See [LICENSE](../../LICENSE).

## Open Questions

- **Image-load telemetry for Rule 4.** Rule 4 (FoundationModels load from unsigned process) requires the sensor to be configured for image load events. Verify under **Policy** > **Agent** that this is enabled in your environment, and confirm what field SentinelOne surfaces to indicate Apple-signed versus third-party signed loaders.
- **STAR rule schema versus the JSON in this repo.** The JSON in `apfelbauer-rules.json` follows the STAR Custom Rules schema as documented. If your console version expects a different schema (the API has evolved across Singularity versions), please file an issue with the import-validation error and the schema your console accepts.
- **Composed-chain correlation.** A single STAR rule that fires on the combination of enablement primitives (DSID read, opt-in write, notifyutil from the same parent within ~10s) would be higher-fidelity than the three individual rules. Deep Visibility supports sequence/correlation. If you write one, share back.

## Deployment

### Via STAR Custom Rules

1. Navigate to **Sentinels** > **STAR Custom Rules** (or **Singularity** > **Custom Rules** depending on console version)
2. Click **Create Rule**
3. For each rule in `apfelbauer-rules.json`:
   - Set the **Rule Name** and **Description** from the JSON
   - Paste the `query` field into the **Deep Visibility Query** box
   - Set **Severity** to match the JSON severity field
   - Set **Rule Type** to **Detection**
   - Under **Scope**, select the appropriate site or account scope
4. Set the **Response** action (recommended: **Detect** for initial deployment, escalate to **Protect** after tuning)
5. Click **Save and Enable**

### Via API

```bash
# Export rules programmatically
for rule in $(jq -c '.[]' apfelbauer-rules.json); do
  curl -X POST "https://<console>.sentinelone.net/web/api/v2.1/cloud-detection/rules" \
    -H "Authorization: ApiToken <token>" \
    -H "Content-Type: application/json" \
    -d "$rule"
done
```

## Query Syntax

Rules use SentinelOne Deep Visibility query language (also called PowerQuery). Key operators:

- `EventType` filters event class (Process Creation, File Modification, etc.)
- `SrcProcName`, `SrcProcCmdLine` match the source process
- `SrcProcParentName` matches the parent process
- `TgtProcName`, `TgtProcCmdLine` match the target/child process
- `IndicatorDescription` is set when the rule fires

## Tuning

- **Rule 5 (Swift non-Xcode parent):** Add exclusions for known CI/CD runners or developer toolchains: `AND SrcProcParentName NOT IN ("fastlane","xcodebuild","xcrun")`
- **Rule 7 (Rapid shell exec):** Adjust the process count or time window based on environment baseline

## Notes

- Deep Visibility retains event data for the configured retention period (default 14 days). Historical queries can surface past attacker activity against this surface.
- Library/framework load events (Rule 4) require the sensor to be configured for image load telemetry. Verify this is enabled under **Policy** > **Agent** settings.
