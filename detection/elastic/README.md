# Elastic Security Detection Rules

Detection rules for Elastic Security (Elastic SIEM) covering the Apple Intelligence enablement primitives and related framework-access patterns described in the project [README](../../README.md).

**Status: untested in a production Elastic Security environment.** I do not have an Elastic stack with live macOS endpoint telemetry to validate these against. The rules are written against the `elastic/detection-rules` repo conventions and ECS field names; they parse and follow the documented format but have not fired against real attack-shaped data in my hands. Review, tune, and test in a non-production space before enabling, and please share back any false-positive patterns or rule-body corrections. **Use at your own risk.** Provided as-is, with no warranty of any kind. See [LICENSE](../../LICENSE).

## Open Questions

- **Elastic Defend library-load coverage.** Rule 4 (FoundationModels load from unsigned process) requires the Elastic Defend policy to capture library/dylib load events under **Integration Policies** > **macOS** > **Library Load Events**. Confirm this is enabled in your environment and that ECS `process.code_signature.subject_name` is populated to reliably distinguish Apple-signed loaders.
- **EQL sequence tuning for Rule 7.** Default `maxspan=10s` with 3+ child processes is a baseline starting point. Tune against your environment and share back what threshold actually fires only on attack-shaped behavior versus developer Swift use.
- **NDJSON round-trip from the TOML.** The TOML in this directory follows `elastic/detection-rules` repo conventions and is intended to be exported to NDJSON via that toolchain (Kibana imports NDJSON, not TOML). If you find a syntax or schema issue in the round-trip, file an issue with the validator error.

## Deployment

### Via the `detection-rules` toolchain (recommended)

Kibana's import endpoint accepts **NDJSON**, not TOML. The TOML in `apfelbauer-rules.toml` follows the [`elastic/detection-rules`](https://github.com/elastic/detection-rules) repo conventions and is intended to be exported to NDJSON via that tooling:

```bash
# Install Elastic's detection-rules toolkit
git clone https://github.com/elastic/detection-rules.git
cd detection-rules && pip install -e .

# View / validate (does not upload)
python -m detection_rules view-rule /path/to/apfelbauer-rules.toml

# Export to NDJSON, then upload via Kibana UI or API
python -m detection_rules export-rules-from-repo --rule-file /path/to/apfelbauer-rules.toml --outfile apfelbauer-rules.ndjson
```

### Via Kibana Detection Rules UI

1. Navigate to **Security** > **Detections** > **Manage detection rules**
2. Click **Import rules**
3. Upload the **NDJSON** produced above (`apfelbauer-rules.ndjson`)
4. Review imported rules and adjust index patterns if needed (default: `logs-endpoint.events.*`)
5. Enable each rule

### Via Detection Rules API

```bash
# Upload the NDJSON (not the TOML) to the import endpoint
curl -X POST "https://<kibana>:5601/api/detection_engine/rules/_import" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@apfelbauer-rules.ndjson"
```

### Via Elastic Agent

Rules use `logs-endpoint.events.*` index patterns, which require:
- **Elastic Agent** with the **Elastic Defend** integration
- macOS endpoint policy with **Process**, **File**, and **Library** event collection enabled

## Rule Format

Rules are written in TOML format compatible with the `elastic/detection-rules` repository structure. Each rule contains:
- `[rule]` metadata (name, description, severity, MITRE mapping)
- `[rule.query]` with EQL or KQL syntax
- `[rule.threat]` with MITRE ATT&CK framework mapping

### Query Languages Used

- **EQL (Event Query Language):** Used for process creation, sequence, and library load rules. Supports process tree analysis and temporal ordering.
- **KQL (Kibana Query Language):** Used for simpler keyword-based rules where EQL sequences are not needed.

## Tuning

- **Rule 5 (Swift non-Xcode parent):** Add process exclusions to the EQL query: `and not process.parent.executable : ("/usr/bin/fastlane", "/opt/homebrew/bin/mint")`
- **Rule 4 (FoundationModels load):** Requires the Elastic Defend policy to collect library/dylib load events. Enable under **Integration Policies** > **macOS** > **Library Load Events**.
- **Rule 7 (Rapid shell exec):** The EQL sequence `maxspan` can be adjusted. Default is 10 seconds with 3+ child processes.

## Notes

- EQL `sequence` rules require the Elastic Common Schema (ECS) fields to be populated by the Elastic Agent.
- Rules target `logs-endpoint.events.*` indices. If using Filebeat or a custom ingest pipeline, adjust index patterns accordingly.
- Elastic's built-in macOS protections may overlap with some rules; these provide specific coverage for the apfelbauer attack pattern.
