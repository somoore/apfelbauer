# Detection Rules

EDR detection rules for the Apple Intelligence enablement primitives and related framework-access patterns described in the project [README](../README.md).

## Status: Starter Artifacts, Untested at Production Scale — Help Wanted

These rules are written from process-tree and preference-write semantics verified on macOS Tahoe 26.5 (build 25F71), but **I do not have a SentinelOne, CrowdStrike, Elastic Security, or Splunk console with live macOS endpoint telemetry to validate them against real fleet data.** Treat them as starter content. Each rule is intentionally simple and tunable. Field-level names, code-signer signal availability, and library-load telemetry coverage all vary by sensor version and policy; the per-vendor READMEs flag the specific points to check. Open questions are at the bottom of this file; if you can answer any of them in your environment, please share what you find.

**Use at your own risk.** Provided as-is, with no warranty of any kind. See [LICENSE](../LICENSE).

## Why Behavioral Detection Is Required

Every component in the chain is an Apple-signed binary:

- `python3` (or `bash`) launches `/usr/bin/swift`
- `swift` loads `FoundationModels.framework` (Apple-signed)
- The framework communicates with `siriinferenced` (Apple system daemon)
- Tool callbacks execute shell commands via standard POSIX APIs

There is nothing unsigned, nothing anomalous at the binary level, and no static payload to signature-match. Behavioral detection (monitoring process trees, preference writes, framework loads, and command sequences) is the only viable approach.

## Detection Rule Categories

| # | Detection | MITRE ATT&CK |
|---|-----------|--------------|
| 1 | `defaults write` to `com.apple.CloudSubscriptionFeatures.optIn` | [T1562.001](https://attack.mitre.org/techniques/T1562/001/) - Disable or Modify Tools |
| 2 | `defaults read MobileMeAccounts` from non-Settings process | [T1552.001](https://attack.mitre.org/techniques/T1552/001/) - Credentials in Files |
| 3 | `notifyutil -p com.apple.CloudSubscriptionFeatures.OptIn.Changed` | [T1562.001](https://attack.mitre.org/techniques/T1562/001/) - Disable or Modify Tools |
| 4 | `FoundationModels.framework` load from unsigned process | [T1059.004](https://attack.mitre.org/techniques/T1059/004/) - Unix Shell |
| 5 | `/usr/bin/swift` from non-Xcode parent (python3, bash, etc.) | [T1059.004](https://attack.mitre.org/techniques/T1059/004/) - Unix Shell |
| 6 | `defaults write` to `com.apple.generativeexperiencesd` | [T1647](https://attack.mitre.org/techniques/T1647/) - Plist File Modification — low-confidence probe indicator: writes succeed but static analysis of `GenerativeExperiencesRuntime` on 25F71 finds no daemon reader for the well-known keys (`disableToolBoxAllowList`, `inputValidation`, `safetyMode`, `bypassTranscriptWriteRedaction`); treat as attacker surface-probing signal, not confirmed compromise. |
| 7 | Rapid sequential shell exec from Swift interpreter | [T1059.004](https://attack.mitre.org/techniques/T1059/004/) - Unix Shell |

## Vendor Coverage Matrix

| Rule | SentinelOne | CrowdStrike | Elastic | Splunk |
|------|:-----------:|:-----------:|:-------:|:------:|
| 1 - CloudSubscriptionFeatures write | Yes | Yes | Yes | Yes |
| 2 - MobileMeAccounts DSID read | Yes | Yes | Yes | Yes |
| 3 - Darwin notification trigger | Yes | Yes | Yes | Yes |
| 4 - FoundationModels unsigned load | Partial | Partial | Yes | Yes |
| 5 - Swift from non-Xcode parent | Yes | Yes | Yes | Yes |
| 6 - generativeexperiencesd write | Yes | Yes | Yes | Yes |
| 7 - Rapid shell exec from Swift | Yes | Yes | Yes | Yes |

**Partial:** SentinelOne and CrowdStrike framework/dylib load telemetry varies by sensor version and configuration. Library load events may require additional sensor configuration to capture.

## Deployment

Each vendor subdirectory contains:
- A `README.md` with vendor-specific deployment instructions
- Detection rules in the vendor's native format

| Vendor | Directory | Format |
|--------|-----------|--------|
| SentinelOne | [sentinelone/](sentinelone/) | JSON (Deep Visibility / STAR queries) |
| CrowdStrike | [crowdstrike/](crowdstrike/) | JSON (Falcon Event Search / FQL queries) |
| Elastic | [elastic/](elastic/) | TOML (Detection rules with EQL/KQL) |
| Splunk | [splunk/](splunk/) | SPL (Saved searches) |

## Coverage Gaps

These rules ship as-is. Defenders should be aware of the following gaps and consider compensating controls:

- **Direct plist writes** to `~/Library/Preferences/com.apple.CloudSubscriptionFeatures.optIn.plist` via `PlistBuddy` or `plutil -replace` are not covered by Rule 1 (which keys on the `defaults` CLI). An attacker that edits the plist file directly will bypass Rule 1. Augment with a File-Integrity-Monitoring rule on that path if your EDR supports it.
- **Python loading `apple-fm-sdk`** (PyPI) is not covered separately; the resulting `siriinferenced` XPC activity is the same as the Swift path, but Rule 4 and Rule 7 may key on Swift-specific signals. Add a process-creation rule for `python3` with `apple_fm_sdk` in the import line or `sys.modules` if your EDR exposes module-load telemetry.
- **Composed-chain detection** (DSID read, opt-in write, notifyutil from the same parent within a short window) is not shipped as a single correlation rule because the windowing/join semantics vary per product. The individual primitives are covered by Rules 1, 2, and 3; tune your SIEM to alert higher when 2+ of those fire from the same parent within ~10s.
- **XPC to `siriinferenced`** from non-Apple-signed callers is typically beyond standard EDR telemetry; this requires an Endpoint Security Framework (ESF) agent.

## Tuning Guidance

- **Rule 5 (Swift from non-Xcode parent):** Developers using Swift CLI tools or Swift scripting will trigger this. Allowlist known developer workflows by parent process path.
- **Rule 7 (Rapid shell exec):** CI/CD pipelines that invoke Swift scripts may match. Tune the time window and command count threshold.
- **Rules 1, 3, 6:** Very low false-positive rate. These preference domains are not written by legitimate software outside of System Settings.

## Open Questions for the Community

If you have a live console for any of the four vendors, these are the high-value tests:

1. **Rule 4 sensor coverage.** Does your CrowdStrike or SentinelOne sensor default policy actually emit `FoundationModels.framework` library-load events for unsigned processes? CrowdStrike's "Apple-signed" code-signer field varies across sensor versions (`SignInfoFlags`, `CertificatePublisher`, per-event signer fields); the Rule 4 query assumes `SignInfoFlags`, which may be wrong in your environment. Verify with a sample `event_simpleName=ProcessRollup2` lookup and report which field your tenant exposes.
2. **CIM field availability for the Splunk rules.** Confirm that the assumed `process_name`, `command_line`, `parent_process_name` CIM-Endpoint fields are populated by whatever forwarder/integration you use. The Splunk rules are written against CIM; if your data is unnormalized, the field references will need translation.
3. **Composed-chain correlation.** The opt-in enablement primitives ship as three separate rules (DSID read, opt-in write, Darwin notification) because the join/window semantics differ per product. The high-fidelity detection is *all three from the same parent within ~10s*. If you have authored a working SIEM correlation rule for this pattern in any of the four vendors, please share.
4. **EQL `sequence` semantics for Rule 7.** The Elastic rule for rapid shell exec from Swift uses `maxspan=10s` with a 3+ child count. Tune against your baseline and report what threshold actually fires only on attack-shaped behavior versus developer use.
5. **Direct plist-write coverage.** The current rules key on `defaults` and `notifyutil` CLI invocations. A `plutil -replace` or direct `PlistBuddy` write would bypass them. If your EDR supports file-integrity monitoring on that path, please share the rule body.
6. **Apple-FM-SDK on Python.** Rule 4 keys on `FoundationModels.framework` library loads from non-Apple-signed processes. The `apple-fm-sdk` Python wrapper produces the same underlying XPC traffic to `siriinferenced` but via the Python interpreter, which IS Apple-signed. What signal does your sensor surface for `python3` importing `apple_fm_sdk` that would distinguish it from any other Python use?

Open an issue or PR with your console version, sensor policy, and the query/result.

## Related

- [`../README.md`](../README.md) — project overview, enablement commands, and why this matters
- [`../mitigations/`](../mitigations/) — MDM configuration profiles (recommended alongside EDR)
