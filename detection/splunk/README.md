# Splunk Detection Rules

Detection rules as Splunk SPL saved searches covering the Apple Intelligence enablement primitives and related framework-access patterns described in the project [README](../../README.md).

**Status: untested in a production Splunk environment.** I do not have a Splunk console with live macOS endpoint telemetry to validate these against. The queries are written against Splunk CIM `Endpoint` fields (see the Data Model Assumptions section below); if your data sources use different field names, the queries will need translation before they fire. Review, tune, and test on a small index before enabling at fleet scale, and please share back any field-name corrections or false-positive patterns. **Use at your own risk.** Provided as-is, with no warranty of any kind. See [LICENSE](../../LICENSE).

## Open Questions

- **CIM field population.** Are the assumed CIM-Endpoint fields (`process_name`, `command_line`, `parent_process_name`, `dest`) actually populated by your forwarder integration? Universal Forwarder + osquery, FDR-into-Splunk, Jamf Protect-into-Splunk all produce different shapes.
- **Rule 6 (`com.apple.generativeexperiencesd` writes).** Flagged in the top-level README as a low-confidence probe indicator. Treat as hunting signal, not high-fidelity detection — the keys are not confirmed as readers on macOS 26.5.
- **Composed-chain correlation.** Rules 1, 2, and 3 together describe the full enablement chain; a single correlation rule that fires only when 2+ of those trigger from the same parent within ~10s would be much higher fidelity. If you write one, share back.

## Deployment

### Prerequisites

- **Splunk Enterprise** or **Splunk Cloud** with macOS endpoint data
- Data sources: Endpoint telemetry from one of:
  - **Splunk Universal Forwarder** with `osquery` or `Sysmon for macOS`
  - **CrowdStrike Falcon Data Replicator (FDR)** ingested into Splunk
  - **SentinelOne DataSet** forwarded to Splunk
  - **Jamf Protect** telemetry forwarded to Splunk
- The `Splunk Common Information Model (CIM)` app is recommended for field normalization

### Import as Saved Searches

1. Navigate to **Settings** > **Searches, Reports, and Alerts**
2. Click **New Report** for each rule
3. Copy the SPL query from `apfelbauer-rules.spl` — **paste one rule at a time, not the whole file.** The file uses `|`-prefixed lines as visual section dividers; in SPL, `|` is the pipe operator, so pasting the whole file into a single search bar will produce a syntax error. Copy from the `index=...` line of one rule down to the next `# ----` divider.
4. Set the schedule (recommended: every 5 minutes for Critical, every 15 minutes for High/Medium)
5. Configure alert actions (email, webhook, notable event in Splunk ES)

### Import via Splunk Enterprise Security

If using Splunk ES:
1. Navigate to **Configure** > **Content** > **Content Management**
2. Create a new **Correlation Search** for each rule
3. Paste the SPL query and set the MITRE ATT&CK annotation
4. Map to the appropriate **Risk** framework risk score

### Via CLI

```bash
# Import saved searches from the SPL file
# Each search is delimited by comment blocks
splunk add saved-search -name "apfelbauer_rule_1" \
  -search "$(grep -A50 'Rule 1:' apfelbauer-rules.spl | grep -v '^|' | head -20)"
```

## Data Model Assumptions

The SPL queries assume the following field names (Splunk CIM `Endpoint` data model):

| Field | Description |
|-------|-------------|
| `process_name` | Name of the executed process |
| `process_path` | Full path to the process binary |
| `process` / `command_line` | Full command line |
| `parent_process_name` | Name of the parent process |
| `parent_process_path` | Full path to the parent process |
| `dest` | Hostname of the endpoint |
| `_time` | Event timestamp |

If your data uses different field names (e.g., CrowdStrike FDR uses `FileName`, `CommandLine`), adjust the field references accordingly.

## Tuning

- **Rule 5 (Swift non-Xcode parent):** Add `NOT parent_process_name IN ("fastlane","mint","tuist","xcrun")` to exclude CI/CD tooling.
- **Rule 7 (Rapid shell exec):** Adjust the `span=10s` window and `count > 2` threshold based on baseline Swift usage in your environment.
- **All rules:** Add `| where dest NOT IN ("build-server-*","ci-runner-*")` to exclude known build infrastructure.
