# MDM Mitigations

Configuration profiles to disable Apple Intelligence features via MDM on managed Macs. Companion to the EDR rules in [`../detection/`](../detection/) and the project [README](../README.md).

## Status: Untested in the Author's Lab — Help Wanted

These profiles are built from Apple's published MDM restriction keys and the `CloudSubscriptionFeatures` plist structure. **I do not have a Jamf, Kandji, Intune, or Workspace ONE lab and have not validated them against a supervised, DEP-enrolled fleet.** Treat them as starter artifacts and testable hypotheses, not deploy-ready policy. Review and modify for your own environment, pilot before production, and please share what you find; the open questions below need answers the broader community is better positioned to produce.

**Use at your own risk.** Provided as-is, with no warranty of any kind. See [LICENSE](../LICENSE).

## What MDM Alone Cannot Do

The Apple Intelligence on/off state can be re-flipped from unprivileged user code. The `defaults write` + `notifyutil` chain in the project [README](../README.md) operates against `~/Library/Preferences/com.apple.CloudSubscriptionFeatures.optIn` in the user defaults domain. The preference dictionary is keyed by the user's account DSID, which is itself stored in user-readable `MobileMeAccounts.plist`, so the keying does not deter a local attacker who can read that file. The DSID-keyed structure is original design (present in iOS 18.6, predating macOS 26.x), not a recent Apple mitigation.

**On unsupervised/non-DEP Macs that's the end of the story:** an MDM-pushed value in the same user-domain preference offers no precedence over a local `defaults write`. The MDM dashboard will show the device as compliant; the device may not be.

**On supervised/DEP-enrolled Macs the answer is unverified.** See the open questions section below. If managed-preference-domain values take precedence over user-domain values for this preference key on supervised devices, MDM holds the state. If they do not, MDM enforcement for this control is fundamentally broken. I have not found a published test result either way.

**Recommendation regardless of supervision state: deploy MDM profiles AND EDR detection rules together.** MDM profiles raise the baseline (the per-feature UI keys do gate Apple's own apps from exposing generative features). EDR rules ([`../detection/`](../detection/)) give you the observability MDM does not: alerts when the opt-in preference is written, when the Darwin notification fires, or when `FoundationModels.framework` is loaded from an unsigned process.

## Open Questions for the Community

These are the testable hypotheses behind the current profiles. If you have the environment to validate any of them, please publish results (issue / PR / blog post / Mastodon, anywhere I can link). Each has a directly actionable outcome.

1. **Managed-domain precedence for `CloudSubscriptionFeatures.optIn`.** On a supervised, DEP-enrolled Mac with the `disable-apple-intelligence.mobileconfig` profile installed and a user signed into an Apple Account: does a local `defaults write com.apple.CloudSubscriptionFeatures.optIn <DSID> -bool true` followed by `notifyutil -p com.apple.CloudSubscriptionFeatures.OptIn.Changed` flip the daemon's view of the state, or does the managed preference win? Test all four major MDMs (Jamf, Kandji, Intune, Workspace ONE); the answer may vary by how each vendor implements the payload.
2. **Per-feature key behavior under direct framework loads.** Do the documented restriction keys (`allowWritingTools` and the rest) actually block a process from loading `FoundationModels.framework` and invoking the model directly, or are they purely UI gates? Static analysis suggests the latter, but a test where an unsigned binary attempts a model inference under an enforcing profile would settle it.
3. **Update-time behavior of the master toggle.** Does an installed MDM profile that disables Apple Intelligence survive a macOS update that re-prompts the user for AI consent? Multiple users have reported the toggle silently re-enabling after updates on unmanaged devices.
4. **Profile precedence under user-defaults override.** If the profile sets `applicationaccess` keys but a user-context process writes directly to those domains, what does the daemon read on its next refresh?
5. **Removing-the-account behavior.** Sign-out and re-sign-in of the Apple Account changes the DSID. Does the managed preference still apply after the DSID changes, or does the device fall back to default (AI on)?

If you can run any of these, please open an issue or PR with your build, MDM, profile state, and command sequence. Confirmed results will be folded into the README and credited.

## What MDM Can Control

Apple provides per-feature restriction keys for Apple Intelligence capabilities:

| Restriction Key | Controls | Reference |
|----------------|----------|-----------|
| `allowAppleIntelligenceReport` | Apple Intelligence Report sharing | Apple MDM documentation |
| `allowGenmoji` | Genmoji creation | Apple MDM documentation |
| `allowImagePlayground` | Image Playground | Apple MDM documentation |
| `allowImageWand` | Image Wand in Notes | Apple MDM documentation |
| `allowWritingTools` | Writing Tools (systemwide) | Apple MDM documentation |
| `allowMailSmartReplies` | Smart Replies in Mail | Apple MDM documentation |
| `allowMailSummary` | Mail message summarization | Apple MDM documentation |
| `allowNotificationsSummary` | Notification summaries | Apple MDM documentation |

These are UI-surface restrictions: they prevent Apple's own apps from exposing generative features to the user. They do **not** prevent any process from loading `FoundationModels.framework` and invoking the on-device LLM directly. No framework-level MDM control exists.

## What MDM Cannot Control

There is **no MDM restriction key** for:

- **FoundationModels framework access.** Any process can load the framework and invoke the on-device LLM regardless of MDM configuration.
- **Tool protocol usage.** Tool callbacks execute in the calling process with no entitlement check.
- **The `com.apple.generativeexperiencesd` preference domain.** The domain is user-writable. Static analysis of the daemon's dylib on macOS 26.5 (25F71) does not confirm any well-known keys (`disableToolBoxAllowList`, `inputValidation`, `safetyMode`, `bypassTranscriptWriteRedaction`) are read by the daemon, so writes to this domain are best treated as a low-confidence attacker-probing signal rather than a confirmed safety-control bypass.
- **The enablement preference itself.** While MDM can set the preference, a user-level `defaults write` can override it on unsupervised devices. Supervised-device precedence is the open question above.

## Available Profiles

Each MDM vendor directory contains:

| Vendor | Directory | Profile |
|--------|-----------|---------|
| Kandji | [kandji/](kandji/) | `disable-apple-intelligence.mobileconfig` |
| Intune | [intune/](intune/) | `disable-apple-intelligence.mobileconfig` |
| JAMF Pro | [jamf/](jamf/) | `disable-apple-intelligence.mobileconfig` |

Each vendor profile has unique `PayloadUUID` values to avoid collisions. **Before deploying to production, regenerate both the `PayloadUUID` _and_ the `PayloadIdentifier`** values (the inner payloads share identifiers like `com.apfelbauer.disable-ai.applicationaccess` across vendors; if you deploy the Kandji and JAMF copies side-by-side without changing them, one will overwrite the other). On macOS, run `uuidgen` for UUIDs and pick a reverse-DNS prefix you own (e.g., `com.acme.macsec.disable-ai.*`) for identifiers.

## Profile Payload

The `.mobileconfig` profiles set managed preferences to disable Apple Intelligence features. The payload targets:

- `com.apple.applicationaccess` — per-feature restriction keys
- `com.apple.CloudSubscriptionFeatures.optIn` — Apple Intelligence enablement preference

## Defense-in-Depth Recommendation

| Layer | Purpose | Limitation |
|-------|---------|------------|
| **MDM Profile** | Disable AI features for normal use | Bypassable on unsupervised devices via the enablement chain in the project README; supervised-device precedence is the open question |
| **EDR Rules** | Detect bypass attempts in real time | Requires endpoint agent |
| **Package Proxy** | Block `apple-fm-sdk` from PyPI | Does not prevent Swift-based access |
| **User Education** | Awareness of unusual `defaults write` / `notifyutil` patterns | Cannot prevent automated attacks |

Deploy all layers for comprehensive coverage. See [`../detection/`](../detection/) for EDR rules.
